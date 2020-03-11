- Contribution Name: Scale Testing
- Implementation Owner:  ['kohsheen1234'](https://github.com/kohsheen1234)
- Start Date: 29-02-2020
- Target Date: 
- RFC PR:
- Linkerd Issue: [linkerd/linkerd2#3895](https://github.com/linkerd/linkerd2/issues/3895)
- Reviewers: 

[summary]: #summary

# Summary
This contribution aims to completely automate the scale test framework, to ensure the scale tests are repeatable by the community. Automating scale tests serve to provide visibility into the proxy's performance and reveal any performance degradation.
# Problem Statement (Step 1)

[problem-statement]: #problem-statement

- Identify performance regression/potential errors with minimal manual intervention.
- Latency in the network communication can be a massive issue. When new features added to Linkerd, it might deteriorate the performance of Linkerd. So, it is important to understand if any new feature which has been added recently has a negative impact on Linkerd's performance. Manually checking every time if there has been a regression in the performance is not a convenient solution. This process can be automated, so that it is very simple to perform scale testing.
- Once this scale testing has been automated,the scale test framework will be able to :
  - Automatically add a sample workload to the cluster.
  - Record cluster,control pane and data plane metrics during the test.
    - Success rate
    - Latency and throughput
    - Dashboard load times
    - `linkerd stat` responsiveness
    - Prometheus cpu/memory usage
  - Report on resource usage, Linkerd performance and potential errors encountered.

# Design proposal (Step 2)

[design-proposal]: #design-proposal

The idea is to create a **Github Bot** that performs Scale and Performance Testing. Here is the procedure that can be followed to achieve the desired automation:

### PART 1 :  
### Installing linkerd along with other resources such as Kubernetes cluster, tainted nodes to run Prometheus etc.
[Part-1]: #Part-1

- Create a script that installs linkerd
  
```sh

#!/bin/bash

# Linkerd install script

# mentioned version here, can be changed to edge version also
release="${1:-stable-2.6.0}"

log "Installing Linkerd version $release"

# To install Linkerd CLI
curl -sL https://run.linkerd.io/install | sh

# Add Linkerd to your path
export PATH=$PATH:$HOME/.linkerd2/bin

# Verify the CLI is installed and running correctly
linkerd version

kubectl create clusterrolebinding cluster-admin-binding-$USER \
   --clusterrole=cluster-admin --user=$(gcloud config get-value account)

# To check that your cluster is configured correctly and ready to install the control plane
linkerd check --pre

linkerd install | kubectl apply -f -

# Validate the intallation
linkerd check

```

- For load testing, the setup requires a very large cluster - at least 32 vCPUs reserved for Linkerd would be recommended. The defaults values are 32vCP and at least 4 nodes.

- For testing stability and e2e behavior in small clusters - 4vCPU per node and 1 node with auto-scaling should work.

- The aim is to make the scale test infractructure runs with [Github Actions](https://github.com/features/actions) on a [Google Kubernetes Engine Cluster](https://cloud.google.com/kubernetes-engine/) or with a single line command locally.

- Set the following environment variables and deploy the cluster.

```sh
  export PROJECT_ID=<google-cloud project-id>
  export CLUSTER_NAME=prombench
  export ZONE=us-east1-b
  export AUTH_FILE=<path to service-account.json>

  ./prombench gke cluster create -a $AUTH_FILE -v PROJECT_ID:$PROJECT_ID \
      -v ZONE:$ZONE -v CLUSTER_NAME:$CLUSTER_NAME -f manifests/cluster.yaml
```

- Script to create a deafult GKE Cluster

```sh
# Creates a standard GKE cluster for testing.

# get default GKE cluster version for zone
function default_gke_version() {
  local zone=${1:?"zone is required"}
  # shellcheck disable=SC2155
  local temp_fname=$(mktemp)

  # shellcheck disable=SC2086
  gcloud container get-server-config --zone "${zone}"  > ${temp_fname} 2>&1
  # shellcheck disable=SC2181
  if [[ $? -ne 0 ]];then
    cat "${temp_fname}"
    exit 1
  fi

  # shellcheck disable=SC2002
  gke_ver=$(cat "${temp_fname}" | grep defaultClusterVersion | awk '{print $2}')
  echo "${gke_ver}"
  rm -rf "${temp_fname}"
}

# Required params
PROJECT_ID=${PROJECT_ID:?"project id is required"}

set +u # Allow referencing unbound variable $CLUSTER
if [[ -z ${CLUSTER} ]]; then
  CLUSTER_NAME=${1:?"cluster name is required"}
else
  CLUSTER_NAME=${CLUSTER}
fi
set -u

# Optional params
ZONE=${ZONE:-us-central1-a}
# specify REGION to create a regional cluster
REGION=${REGION:-}

# Check if zone or region is valid
if [[ -n "${REGION}" ]]; then
  if [[ -z "$(gcloud compute regions list --filter="name=('${REGION:-}')" --format="csv[no-heading](name)")" ]]; then
    echo "No such region: ${REGION:-}. Exiting."
    exit 1
  fi
  ZONE="${REGION}"
else
  if [[ -z "$(gcloud compute zones list --filter="name=('${ZONE}')" --format="csv[no-heading](name)")"  ]]; then
    echo "No such zone: ${ZONE}. Exiting."
    exit 1
  fi  
fi

# Specify GCP_SA to create and use a specific service account.
# Default is to use same name as the cluster - each cluster can have different
# IAM permissions. This also enables workloadIdentity, which is recommended for GCP
GCP_SA=${GCP_SA:-$CLUSTER_NAME}
GCP_CTL_SA=${GCP_CTL_SA:-${CLUSTER_NAME}-control}

# Sizing
DISK_SIZE=${DISK_SIZE:-64}
MACHINE_TYPE=${MACHINE_TYPE:-n1-standard-32}
MIN_NODES=${MIN_NODES:-"4"}
MAX_NODES=${MAX_NODES:-"70"}
IMAGE=${IMAGE:-"COS"}
MAXPODS_PER_NODE=100

# Labels and version
ISTIO_VERSION=${ISTIO_VERSION:-master}

DEFAULT_GKE_VERSION=$(default_gke_version "${ZONE}")
# shellcheck disable=SC2181
if [[ $? -ne 0 ]];then
  echo "${DEFAULT_GKE_VERSION}"
  exit 1
fi

GKE_VERSION=${GKE_VERSION-${DEFAULT_GKE_VERSION}}

SCOPES="${SCOPES_FULL}"

# A label cannot have "." in it, replace "." with "_"
# shellcheck disable=SC2001
ISTIO_VERSION=$(echo "${ISTIO_VERSION}" | sed 's/\./_/g')

function gc() {
  # shellcheck disable=SC2236
  if [[ -n "${REGION:-}" ]];then
    ZZ="--region=${REGION}"
  else
    ZZ="--zone=${ZONE}"
  fi

  SA=""
  # shellcheck disable=SC2236
  if [[ -n "${GCP_SA:-}" ]];then
    SA=("--identity-namespace=${PROJECT_ID}.svc.id.goog" "--service-account=${GCP_SA}@${PROJECT_ID}.iam.gserviceaccount.com" "--workload-metadata-from-node=EXPOSED")
  fi

  # shellcheck disable=SC2048
  # shellcheck disable=SC2086
  echo gcloud $* "${ZZ}" "${SA[@]}"

  # shellcheck disable=SC2236
  set +u
  if [[ -n "${DRY_RUN:-}" ]];then
    return
  fi
  set -u

  # shellcheck disable=SC2086
  # shellcheck disable=SC2048
  gcloud $* "${ZZ}" "${SA[@]}"
}

NETWORK_SUBNET="--create-subnetwork name=${CLUSTER_NAME}-subnet"
# shellcheck disable=SC2236
set +u
if [[ -n "${USE_SUBNET:-}" ]];then
  NETWORK_SUBNET="--network ${USE_SUBNET}"
fi
set -u

ADDONS="HorizontalPodAutoscaling"
# shellcheck disable=SC2236
set +u
if [[ -n "${ISTIO_ADDON:-}" ]];then
  ADDONS+=",Istio"
fi
set -u

# Export CLUSTER_NAME so it will be set for the create_sa.sh script, which will
# create set up service accounts
export CLUSTER_NAME
mkdir -p "${WD}/tmp/${CLUSTER_NAME}"
"${WD}/create_sa.sh" "${GCP_SA}" "${GCP_CTL_SA}"

# shellcheck disable=SC2086
# shellcheck disable=SC2046
if [[ "$(gcloud beta container --project "${PROJECT_ID}" clusters list --filter=name="${CLUSTER_NAME}" --format='csv[no-heading](name)')" ]]; then
  echo "Cluster with this name already created, skipping creation and rerunning init"
else
  gc beta container \
    --project "${PROJECT_ID}" \
    clusters create "${CLUSTER_NAME}" \
    --no-enable-basic-auth --cluster-version "${GKE_VERSION}" \
    --issue-client-certificate \
    --machine-type "${MACHINE_TYPE}" --image-type ${IMAGE} --disk-type "pd-standard" --disk-size "${DISK_SIZE}" \
    --scopes "${SCOPES}" \
    --num-nodes "${MIN_NODES}" --enable-autoscaling --min-nodes "${MIN_NODES}" --max-nodes "${MAX_NODES}" --max-pods-per-node "${MAXPODS_PER_NODE}" \
    --enable-stackdriver-kubernetes \
    --enable-ip-alias \
    --metadata disable-legacy-endpoints=true \
    ${NETWORK_SUBNET} \
    --default-max-pods-per-node "${MAXPODS_PER_NODE}" \
    --addons "${ADDONS}" \
    --enable-network-policy \
    --workload-metadata-from-node=EXPOSED \
    --enable-autoupgrade --enable-autorepair \
    --labels csm=1,test-date=$(date +%Y-%m-%d),version=${ISTIO_VERSION},operator=user_${USER}
fi

NETWORK_NAME=$(basename "$(gcloud container clusters describe "${CLUSTER_NAME}" --project "${PROJECT_ID}" --zone="${ZONE}" \
    --format='value(networkConfig.network)')")
SUBNETWORK_NAME=$(basename "$(gcloud container clusters describe "${CLUSTER_NAME}" --project "${PROJECT_ID}" \
    --zone="${ZONE}" --format='value(networkConfig.subnetwork)')")

# Getting network tags is painful. Get the instance groups, map to an instance,
# and get the node tag from it (they should be the same across all nodes -- we don't
# know how to handle it, otherwise).
INSTANCE_GROUP=$(gcloud container clusters describe "${CLUSTER_NAME}" --project "${PROJECT_ID}" --zone="${ZONE}" --format='flattened(nodePools[].instanceGroupUrls[].scope().segment())' |  cut -d ':' -f2 | head -n1 | sed -e 's/^[[:space:]]*//' -e 's/::space:]]*$//')
INSTANCE_GROUP_ZONE=$(gcloud compute instance-groups list --filter="name=(${INSTANCE_GROUP})" --format="value(zone)" | sed 's|^.*/||g')
sleep 1
INSTANCE=$(gcloud compute instance-groups list-instances "${INSTANCE_GROUP}" --project "${PROJECT_ID}" \
    --zone="${INSTANCE_GROUP_ZONE}" --format="value(instance)" --limit 1)
NETWORK_TAGS=$(gcloud compute instances describe "${INSTANCE}" --zone="${INSTANCE_GROUP_ZONE}" --project "${PROJECT_ID}" --format="value(tags.items)")


NEGZONE=""
if [[ -n "${REGION}" ]]; then
  NEGZONE="region = ${REGION}"
else
  NEGZONE="local-zone = ${ZONE}"
fi

CONFIGMAP_NEG=$(cat <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: gce-config
  namespace: kube-system
data:
  gce.conf: |
    [global]
    token-url = nil
    # Your cluster's project
    project-id = ${PROJECT_ID}
    # Your cluster's network
    network-name =  ${NETWORK_NAME}
    # Your cluster's subnetwork
    subnetwork-name = ${SUBNETWORK_NAME}
    # Prefix for your cluster's IG
    node-instance-prefix = gke-${CLUSTER_NAME}
    # Network tags for your cluster's IG
    node-tags = ${NETWORK_TAGS}
    # Zone the cluster lives in
    ${NEGZONE}
EOF
)


CONFIGMAP_GALLEY=$(cat <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: istiod-asm
  namespace: istio-system
data:
  galley.json: |
      {
      "EnableServiceDiscovery": true,
      "SinkAddress": "meshconfig.googleapis.com:443",
      "SinkAuthMode": "GOOGLE",
      "ExcludedResourceKinds": ["Pod", "Node", "Endpoints"],
      "sds-path": "/etc/istio/proxy/SDS",
      "SinkMeta": ["project_id=${PROJECT_ID}"]
      }
  PROJECT_ID: ${PROJECT_ID}
  GOOGLE_APPLICATION_CREDENTIALS: /var/secrets/google/key.json
  ISTIOD_ADDR: istiod-asm.istio-system.svc:15012
  WEBHOOK: istiod-asm
  AUDIENCE: ${PROJECT_ID}.svc.id.goog
  trustDomain: ${PROJECT_ID}.svc.id.goog
  gkeClusterUrl: https://container.googleapis.com/v1/projects/${PROJECT_ID}/locations/${ZONE}/clusters/${CLUSTER_NAME}
EOF
)

export KUBECONFIG="${WD}/tmp/${CLUSTER_NAME}/kube.yaml"
gcloud container clusters get-credentials "${CLUSTER_NAME}" --zone "${ZONE}"

# The gcloud key create command requires you dump its service account
# credentials to a file. Let that happen, then pull the contents into a varaible
# and delete the file.
CLOUDKEY=""
if [[ "${CLUSTER_NAME}" != "" ]]; then 
  if ! kubectl -n kube-system get secret google-cloud-key >/dev/null 2>&1 || ! kubectl -n istio-system get secret google-cloud-key > /dev/null 2>&1; then
    gcloud iam service-accounts keys create "${WD}/tmp/${CLUSTER_NAME}/${CLUSTER_NAME}-cloudkey.json" --iam-account="${GCP_CTL_SA}"@"${PROJECT_ID}".iam.gserviceaccount.com
    # Read from the named pipe into the CLOUDKEY variable
    CLOUDKEY=$(cat "${WD}/tmp/${CLUSTER_NAME}/${CLUSTER_NAME}-cloudkey.json")
    # Clean up
    rm "${WD}/tmp/${CLUSTER_NAME}/${CLUSTER_NAME}-cloudkey.json"
  fi
fi

if ! kubectl get clusterrolebinding cluster-admin-binding > /dev/null 2>&1; then 
  kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user="$(gcloud config get-value core/account)"
fi

# Update the cluster with the GCP-specific configmaps
if ! kubectl -n kube-system get secret google-cloud-key > /dev/null 2>&1; then 
  kubectl -n kube-system create secret generic google-cloud-key  --from-file key.json=<(echo "${CLOUDKEY}")
fi
kubectl -n kube-system apply -f <(echo "${CONFIGMAP_NEG}")

if ! kubectl get ns istio-system > /dev/null; then
  kubectl create ns istio-system
fi
if ! kubectl -n istio-system get secret google-cloud-key > /dev/null 2>&1; then 
  kubectl -n istio-system create secret generic google-cloud-key  --from-file key.json=<(echo "${CLOUDKEY}")
fi
kubectl -n istio-system apply -f <(echo "${CONFIGMAP_GALLEY}")
```

- Can be used while trigerring scale test by a Github Action i.e Comment in our case.
  
```yaml
  - name: Create GKE cluster
  # linkerd/linkerd2-action-gcloud@v1.0.1
  uses: linkerd/linkerd2-action-gcloud@308c4df
  with:
    cloud_sdk_service_account_key: ${{ secrets.CLOUD_SDK_SERVICE_ACCOUNT_KEY }}
    gcp_project: ${{ secrets.GCP_PROJECT }}
    gcp_zone: ${{ secrets.GCP_ZONE }}
    create: true
    name: testing-${{ steps.install_cli.outputs.tag }}-${{ github.run_id }}
```

```sh
  ifeq ($(AUTH_FILE),)
  AUTH_FILE = /etc/serviceaccount/service-account.json
  endif

  .PHONY: deploy clean
  deploy: nodepool_create resource_apply

  nodepool_create:
    $(PROMBENCH_CMD) gke nodepool create -a ${AUTH_FILE} \
        -v ZONE:${ZONE} -v PROJECT_ID:${PROJECT_ID} -v CLUSTER_NAME:${CLUSTER_NAME} -v PR_NUMBER:${PR_NUMBER} \
        -f manifests/prombench/nodepools.yaml

resource_apply:
    $(PROMBENCH_CMD) gke resource apply -a ${AUTH_FILE} \
        -v ZONE:${ZONE} -v PROJECT_ID:${PROJECT_ID} -v CLUSTER_NAME:${CLUSTER_NAME} \
        -v PR_NUMBER:${PR_NUMBER} -v RELEASE:${RELEASE} -v DOMAIN_NAME:${DOMAIN_NAME} \
        -v GITHUB_ORG:${GITHUB_ORG} -v GITHUB_REPO:${GITHUB_REPO} \
        -f manifests/prombench/benchmark

# NOTE: required because namespace and cluster-role are not part of the created nodepools
resource_delete:
    $(PROMBENCH_CMD) gke resource delete -a ${AUTH_FILE} \
        -v ZONE:${ZONE} -v PROJECT_ID:${PROJECT_ID} -v CLUSTER_NAME:${CLUSTER_NAME} -v PR_NUMBER:${PR_NUMBER} \
        -f manifests/prombench/benchmark/1c_cluster-role-binding.yaml \
        -f manifests/prombench/benchmark/1a_namespace.yaml

nodepool_delete:
    $(PROMBENCH_CMD) gke nodepool delete -a ${AUTH_FILE} \
        -v ZONE:${ZONE} -v PROJECT_ID:${PROJECT_ID} -v CLUSTER_NAME:${CLUSTER_NAME} -v PR_NUMBER:${PR_NUMBER} \
        -f manifests/prombench/nodepools.yaml

all_nodepools_running:
    $(PROMBENCH_CMD) gke nodepool check-running -a ${AUTH_FILE} \
        -v ZONE:${ZONE} -v PROJECT_ID:${PROJECT_ID} \
        -v CLUSTER_NAME:${CLUSTER_NAME} -v PR_NUMBER:${PR_NUMBER} \
        -f manifests/prombench/nodepools.yaml

all_nodepools_deleted:
    $(PROMBENCH_CMD) gke nodepool check-deleted -a ${AUTH_FILE} \
        -v ZONE:${ZONE} -v PROJECT_ID:${PROJECT_ID} \
        -v CLUSTER_NAME:${CLUSTER_NAME} -v PR_NUMBER:${PR_NUMBER} \
        -f manifests/prombench/nodepools.yaml
```

- Function can be used to create clusters along with resources.

```go
  func K8Clusters {
    g := gke.New()

    k8sGKE := app.Command("gke", `Google container engine provider - https://cloud.google.com/kubernetes-engine/`).
        Action(g.NewGKEClient)
    k8sGKE.Flag("auth", "json authentication for the project. Accepts a filepath or an env variable that inlcudes tha json data. If not set the tool will use the GOOGLE_APPLICATION_CREDENTIALS env variable (export GOOGLE_APPLICATION_CREDENTIALS=service-account.json). https://cloud.google.com/iam/docs/creating-managing-service-account-keys.").
        PlaceHolder("service-account.json").
        Short('a').
        StringVar(&g.Auth)
    k8sGKE.Flag("file", "yaml file or folder  that describes the parameters for the object that will be deployed.").
        Required().
        Short('f').
        ExistingFilesOrDirsVar(&g.DeploymentFiles)
    k8sGKE.Flag("vars", "When provided it will substitute the token holders in the yaml file. Follows the standard golang template formating - {{ .hashStable }}.").
        Short('v').
        StringMapVar(&g.DeploymentVars)

    // Cluster operations.
    k8sGKECluster := k8sGKE.Command("cluster", "manage GKE clusters").
        Action(g.GKEDeploymentsParse)
    k8sGKECluster.Command("create", "gke cluster create -a service-account.json -f FileOrFolder").
        Action(g.ClusterCreate)
    k8sGKECluster.Command("delete", "gke cluster delete -a service-account.json -f FileOrFolder").
        Action(g.ClusterDelete)

    // Cluster node-pool operations
    k8sGKENodePool := k8sGKE.Command("nodepool", "manage GKE clusters nodepools").
        Action(g.GKEDeploymentsParse)
    k8sGKENodePool.Command("create", "gke nodepool create -a service-account.json -f FileOrFolder").
        Action(g.NodePoolCreate)
    k8sGKENodePool.Command("delete", "gke nodepool delete -a service-account.json -f FileOrFolder").
        Action(g.NodePoolDelete)
    k8sGKENodePool.Command("check-running", "gke nodepool check-running -a service-account.json -f FileOrFolder").
        Action(g.AllNodepoolsRunning)
    k8sGKENodePool.Command("check-deleted", "gke nodepool check-deleted -a service-account.json -f FileOrFolder").
        Action(g.AllNodepoolsDeleted)

    // K8s resource operations.
    k8sGKEResource := k8sGKE.Command("resource", `Apply and delete different k8s resources - deployments, services, config maps etc.Required variables -v PROJECT_ID, -v ZONE: -west1-b -v CLUSTER_NAME`).
        Action(g.NewK8sProvider).
        Action(g.K8SDeploymentsParse)
    k8sGKEResource.Command("apply", "gke resource apply -a service-account.json -f manifestsFileOrFolder -v PROJECT_ID:test -v ZONE:europe-west1-b -v CLUSTER_NAME:test -v hashStable:COMMIT1 -v hashTesting:COMMIT2").
        Action(g.ResourceApply)
    k8sGKEResource.Command("delete", "gke resource delete -a service-account.json -f manifestsFileOrFolder -v PROJECT_ID:test -v ZONE:europe-west1-b -v CLUSTER_NAME:test -v hashStable:COMMIT1 -v hashTesting:COMMIT2").
        Action(g.ResourceDelete)

}

```

### STEP 2 :  
### Record cluster, control plane and data plane metrics during the test.
[Part-2]: #Part-2
# Deliverables

# Non Goals

# Prior art

[prior-art]: #prior-art

- Scale and performance testing is included in other service mesh. 
  - For instance, let's consider Istio https://istio.io/docs/ops/deployment/performance-and-scalability/. 
  - Github link - https://github.com/istio/tools/tree/release-1.4/perf
- Current scale-test of linkerd - https://github.com/linkerd/linkerd2/blob/master/bin/test-scale 

# Unresolved questions

[unresolved-questions]: #unresolved-questions



# Future possibilities

[future-possibilities]: #future-possibilities
