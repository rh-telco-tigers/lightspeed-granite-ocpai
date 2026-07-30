# Introduction

Self hosting a LLM model for OpenShift Lightspeed is possible. This set of files and readme will walk you through the process of deploying OpenShift AI, hosting a Graphite41 model and making it available to OpenShift LIghtspeed. 

## Requirements

* OpenShift 4 cluster - tested with OpenShift 4.22.6
* Hardware GPU - in order to have a useable model , a GPU card with at LEAST 16GB of ram 
* GPU Operator configured for your platform. This doc will discuss using NVIDIA GPUs, but others will work as well

### Installing OpenShift AI

We will be installing OpenShift AI version 3.4. This can be installed via the GUI or from the command line:

### Installing from the OpenShift UI

To install OpenShift AI from the UI, follow these steps:
1. log into OpenShift with cluster administrator privledges
2. Select "Ecosystem"->"Catalog"
3. Search for "OpenShift AI"
4. Select Install
5. Select the 3.4 version
6. Click Install

### Installing from the command line

You can also install OpenShift AI from the command line. Follow the steps listed below

1. log into the cluster with `oc login`
2. run the following command `oc apply -f openshift-ai/operator`
> Note: if you see any failures re-apply the missed files

#### Creating hardware profile for your GPU

Before moving to the next step and hosting the Granite model, we need to create a `HardwareProfile` which defines what GPU hardware you have available. The following steps assume you have already installed/configured your GPU operator. For this document we will assume NVIDIA hardware.

Start by getting your GPU hardware available in your cluster. The easiest way to do this is to look at the Allocatable Resources for each node in your cluster. Look for the identifier for your GPU hardware. For example:

```
oc describe node ec-eb-b8-97-af-ac
Name:               ec-eb-b8-97-af-ac
Roles:              control-plane,master,worker
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
...
Capacity:
  cpu:                56
  ephemeral-storage:  585217732Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             263974856Ki
  nvidia.com/gpu:     4
  pods:               250
```

>NOTE: In the above output, `nvidia.com/gpu: 4` shows that there are 4 NVIDIA GPUs available

Using the file `openshift-ai/postconfig/04_hardware_profile.yaml` update the displayname and identifier to match what you have in your cluster.

Apply the hardwareprofile using

`oc apply -f openshift-ai/postconfig/04_hardware_profile.yaml`

> NOTE If you change the name of the hardware profile, keep track of this, as we will need it later on.
## Preparing the Granite41 model (optional)

There are multiple ways to get a copy of the Granite41 model for hosting:

1. **Deploy directly from Hugging Face** — OpenShift AI pulls the model at deploy time. Skip this section and go to [Hosting our model with OpenShift AI](#hosting-our-model-with-openshift-ai).
2. **Build a container image** — download the model locally, package it, and push to a registry (steps below).
3. **Upload to S3** — download the model locally and sync it to an S3-compatible bucket (steps below).

If you choose option 2 or 3, start by installing the Hugging Face CLI. See [Install the Hugging Face CLI](https://huggingface.co/docs/huggingface_hub/main/en/installation#install-the-hugging-face-cli) for details.

Download the model:

```sh
hf auth login
hf download ibm-granite/granite-4.1-3b --local-dir ./granite-41/model --exclude "*.bin" --exclude "*.gguf" --exclude "original/*"
```

### Building the Container Image
```
export IMAGE_REPO=registry.xphyrlab.net/models/granite
export TAG=4.1-3b
podman build \
  --platform linux/amd64 \
  -t "${IMAGE_REPO}:${TAG}" \
  -f "./granite-41/Containerfile" \
  "./granite-41"
```

Push your container file to your local registry. We will need this later:

```sh
podman push "${IMAGE_REPO}:${TAG}"
```

### Uploading to S3 Bucket

It is also possible to use S3 bucket storage to upload the model we just pulled from HuggingFace. You will need a S3 bucket and the aws-cli tool to upload the data:

```
export AWS_ACCESS_KEY_ID=8tMRAZKZ2ro44nwInOgh
export AWS_SECRET_ACCESS_KEY=2asLPIgbPn8sPDB4T0mOsd4kl9iBy8e6Plr3rXRw
export ENDPOINT_URL=https://s3.xphyrlab.net:9445
export MODELS_BUCKET=models
export MODEL_NAME=granite413b
aws --endpoint-url=${ENDPOINT_URL} s3 sync ./granite-41/model/ s3://${MODELS_BUCKET}/${MODEL_NAME}/
```

>Note due to the very large size of these files, you may run into issues with uploading to S3. You can use the following additional commands to tweak the upload process:
`export AWS_MAX_ATTEMPTS=10`
`aws configure set default.s3.multipart_chunksize 64MB`

## Hosting our model with OpenShift AI

With OpenShift AI installed, you can host the model either by applying manifests from the CLI or by using the OpenShift AI dashboard UI. Prefer **Option 1 (Hugging Face)** when the cluster can reach Hugging Face. Use Option 2 if you prepared a container image or S3 upload earlier.

Manifests live under `openshift-ai/model-hosting/`.

### Option 1: Deploy directly from Hugging Face

This path uses:

| File | Purpose |
|------|---------|
| `namespace.yml` | Creates the `llm-serving` namespace |
| `hf-secret.yaml` | Stores your Hugging Face token as `HF_TOKEN` |
| `servingruntime.yaml` | vLLM NVIDIA GPU ServingRuntime for KServe |
| `inference_huggingface.yaml` | InferenceService that pulls `hf://ibm-granite/granite-4.1-3b` |

You need a [Hugging Face access token](https://huggingface.co/settings/tokens) with permission to download `ibm-granite/granite-4.1-3b`. The cluster must be able to reach Hugging Face on the network.

#### Update inference instance to use hardware profile

Edit the `servingruntime.yaml` file and update the `opendatahub.io/hardware-profile-name` to match the name of the hardware profile you created earlier.

#### Via the command line (manifests)

1. Log into the cluster with `oc login`.
2. Create the namespace:

```sh
oc apply -f openshift-ai/model-hosting/namespace.yml
```

3. Edit `openshift-ai/model-hosting/hf-secret.yaml` and replace `<your-hf-token>` with your Hugging Face token, then apply it:

```sh
oc apply -f openshift-ai/model-hosting/hf-secret.yaml
```

4. Apply the ServingRuntime:

```sh
oc apply -f openshift-ai/model-hosting/servingruntime.yaml -n llm-serving
```

5. Apply the Hugging Face InferenceService:

```sh
oc apply -f openshift-ai/model-hosting/inference_huggingface.yaml
```

6. Wait for the deployment to become ready:

```sh
oc get inferenceservice granite-41-3b -n llm-serving
oc get pods -n llm-serving -w
```

The InferenceService uses `storageUri: hf://ibm-granite/granite-4.1-3b` and injects `HF_TOKEN` from the `hf-secret` secret so vLLM can authenticate to Hugging Face when downloading the model.

#### Via the OpenShift AI UI

1. Log into the OpenShift AI dashboard with cluster administrator (or project-admin) privileges.
2. Create a data science project named `llm-serving` (or select it if it already exists). This matches the namespace used by the manifests.
3. Create a secret in OpenShift for Hugging Face authentication (either in the OpenShift console under **Workloads → Secrets** in `llm-serving`, or via `oc`):
   - Name: `hf-secret`
   - Key: `HF_TOKEN`
   - Value: your Hugging Face access token
4. Ensure a vLLM NVIDIA GPU ServingRuntime is available in the project. If it is not listed yet, apply `servingruntime.yaml` from the CLI (Option 1 command-line step 4), or import that YAML from the OpenShift console (**+ → Import YAML**).
5. In your project, open the **Models** tab and click **Deploy model**.
6. Fill in the deployment form:
   - **Model deployment name**: e.g. `granite-41-3b`
   - **Model type**: `Generative AI`
   - **Serving runtime**: `vLLM NVIDIA GPU ServingRuntime for KServe` (or `vllm-cuda-runtime`)
   - **Hardware profile**: match your GPU hardware profile (the sample manifest requests 1 GPU, 4 CPU, 32Gi–48Gi memory)
   - **Model location**: choose a **URI** connection type and set the URI to:

      ```text
      hf://ibm-granite/granite-4.1-3b
      ```
7. Select **Make model deployment available through an external route**.
8. Enable token authentication by selecting **Require token authentication**.
9. Under configuration parameters, add an environment variable so the runtime can authenticate to Hugging Face:
   - Name: `HF_TOKEN`
   - Value from: secret `hf-secret`, key `HF_TOKEN`
10. Add custom runtime arguments, for example:
   - `--max-model-len=15000`
   - `--max-num-seqs=4`
   - `--gpu-memory-utilization=0.95`
   - `--enforce-eager`
   - `--enable-auto-tool-choice`
   - `--tool-call-parser=granite4`
11. Click **Deploy** and wait until the model status shows as ready. The first pull from Hugging Face can take several minutes.

### Option 2: Deploy from a container image or S3

Use this path only if you built a ModelCar/container image or uploaded the model to S3 in the preparation steps above (when Hugging Face direct deploy is not suitable). Adjust `openshift-ai/model-hosting/inferenceservice.yaml` (storage URI / connection details, image pull secrets, and resources) to match your registry or S3 connection before applying.

#### Hosting the Model via the UI

1. In the OpenShift AI dashboard, open (or create) the `llm-serving` project.
2. Open the **Models** tab and click **Deploy model**.
3. Select the vLLM NVIDIA GPU ServingRuntime and configure replicas, server size, and accelerator for your GPU.
4. For the model location:
   - **Container / OCI**: create a URI connection and set the URI to your image (for example `oci://registry.example.com/models/granite:4.1-3b`).
   - **S3**: select or create an S3 connection pointing at your bucket, and set the path to the model prefix (for example `granite413b`).
5. Configure optional serving arguments and authentication as needed, then click **Deploy**.

#### Hosting the Model via the command line

1. Ensure the `llm-serving` namespace and ServingRuntime exist (same as Option 1 steps 2 and 4).
2. Update `openshift-ai/model-hosting/inferenceservice.yaml` for your storage location and credentials.
3. Apply the InferenceService:

```sh
oc apply -f openshift-ai/model-hosting/inferenceservice.yaml
```

4. Watch the pods until the model is ready:

```sh
oc get pods -n llm-serving -w
```

### Testing the model

Before configuring LightSpeed on an OCP cluster, lets test to ensure that the model is running properly

```sh
# SMOKE TEST the model directly before touching OLS:
POD=$(oc get pod -n llm-serving -o jsonpath='{.items[0].metadata.name}')
oc exec -n llm-serving "$POD" -c kserve-container -- \
  curl -s http://localhost:8080/v1/models | python3 -m json.tool
# Should return a model with id "granite-41-3b"

oc exec -n llm-serving "$POD" -c kserve-container -- \
  curl -s -X POST http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "granite-41-3b",
    "messages": [{"role": "user", "content": "What is a Kubernetes Deployment in one sentence?"}],
    "max_tokens": 100
  }' | python3 -m json.tool
# Should return a coherent answer. If this fails, OLS will too —
# debug here first.
```

### Expose AI Model for external access

```
$oc get svc -n llm-serving`
NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
granite-41-3b-metrics     ClusterIP   172.30.165.244   <none>        8080/TCP   8m19s
granite-41-3b-predictor   ClusterIP   None             <none>        80/TCP     8m19s
```

`oc create route edge --service=granite-41-3b-predictor`

```
oc get route
NAME                      HOST/PORT                                                   PATH   SERVICES                  PORT   TERMINATION   WILDCARD
granite-41-3b-predictor   granite-41-3b-predictor-llm-serving.apps.sno.xphyrlab.net          granite-41-3b-predictor   http   edge          None
```

## Installing OpenShift LightSpeed

On the cluster(s) you want to run OpenShift LightSpeed on

To install OpenShift LightSpeed from the UI, follow these steps:
1. log into OpenShift with cluster administrator privledges
2. Select "Ecosystem"->"Catalog"
3. Search for "OpenShift LightSpeed"
4. Select Install
5. Select the latest version
6. Click Install