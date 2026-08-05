# Hosting Granite 4.1-3B for OpenShift LightSpeed

This tutorial walks you through deploying OpenShift AI, hosting an IBM Granite 4.1-3B model, and connecting OpenShift LightSpeed to that model.

## Requirements

* OpenShift 4 cluster (tested with OpenShift 4.22.6)
* GPU with at least 16 GB of VRAM
* GPU Operator configured for your platform (this guide uses NVIDIA; other vendors work similarly)

---

## 1. Install OpenShift AI

Install OpenShift AI 3.4 from the UI or the CLI.

### From the OpenShift UI

1. Log into OpenShift with cluster-administrator privileges.
2. Go to **Ecosystem → Catalog**.
3. Search for **OpenShift AI**.
4. Select version **3.4** and click **Install**.

### From the command line

1. Log into the cluster with `oc login`.
2. Apply the operator manifests:

```sh
oc apply -f openshift-ai/operator
```

> If some resources fail on the first pass (for example, the DataScienceCluster before the operator is ready), re-apply until all succeed.

3. Confirm the DataScienceCluster is ready:

```sh
oc get datasciencecluster -A
oc get pods -n redhat-ods-applications
```

### Create a hardware profile

Before hosting the model, create a `HardwareProfile` that matches your GPU. This assumes the GPU Operator is already installed.

1. Find the GPU resource identifier on a node (look under **Allocatable** / **Capacity**):

```sh
oc describe node <node-name>
```

Example output:

```text
Capacity:
  ...
  nvidia.com/gpu:     4
```

> Here, `nvidia.com/gpu` is the identifier and `4` is the count.

2. Edit `openshift-ai/postconfig/hardware_profile.yaml` so the display name and GPU `identifier` match your cluster.
3. Apply it:

```sh
oc apply -f openshift-ai/postconfig/hardware_profile.yaml
```

> If you rename the hardware profile, note the new name — you will reference it in the InferenceService later.

---

## 2. Prepare the model (optional)

You can serve Granite 4.1-3B in three ways:

1. **Pull from Hugging Face at deploy time** (recommended when the cluster can reach Hugging Face) — skip this section and go to [Host the model](#3-host-the-model-with-openshift-ai).
2. **Build a container image** (ModelCar) — download locally, package, and push to a registry.
3. **Upload to S3** — download locally and sync to an S3-compatible bucket.

For options 2 and 3, install the [Hugging Face CLI](https://huggingface.co/docs/huggingface_hub/main/en/installation#install-the-hugging-face-cli), then download the model:

```sh
hf auth login
hf download ibm-granite/granite-4.1-3b \
  --local-dir ./granite-41/model \
  --exclude "*.bin" \
  --exclude "*.gguf" \
  --exclude "original/*"
```

### Build a container image

If you would like to host the model as a container, follow these steps to build the container and then push to your local registry.

```sh
export IMAGE_REPO=registry.example.com/models/granite
export TAG=4.1-3b

podman build \
  --platform linux/amd64 \
  -t "${IMAGE_REPO}:${TAG}" \
  -f "./granite-41/Containerfile" \
  "./granite-41"

podman push "${IMAGE_REPO}:${TAG}"
```

### Upload to S3

If you would like to host the model in an S3 bucket, you need an S3-compatible bucket and the AWS CLI. Follow these steps to upload the model to your S3 storage:

```sh
export AWS_ACCESS_KEY_ID=<your-access-key>
export AWS_SECRET_ACCESS_KEY=<your-secret-key>
export ENDPOINT_URL=https://s3.example.com:9445
export MODELS_BUCKET=models
export MODEL_NAME=granite413b

aws --endpoint-url="${ENDPOINT_URL}" \
  s3 sync ./granite-41/model/ "s3://${MODELS_BUCKET}/${MODEL_NAME}/"
```

If you have issues with the S3 uploads, the following additional settings often help:

```sh
export AWS_MAX_ATTEMPTS=10
aws configure set default.s3.multipart_chunksize 64MB
```

---

## 3. Host the model with OpenShift AI

With OpenShift AI installed, and a the source of our model determined, we can now move onto hosting the model in OpenShift AI.  Three sets of example manifests are available:

* pull from Huggingface
* pull from Container Registry
* pull from S3 bucket

Manifests live under `openshift-ai/model-hosting/`. Prefer **Option A (Hugging Face)** when the cluster can reach Hugging Face. Use **Option B** if you prepared a container image or S3 upload above.

### Option A: Deploy from Hugging Face

| File | Purpose |
|------|---------|
| `namespace.yml` | Creates the `llm-serving` namespace |
| `hf-secret.yaml` | Stores your Hugging Face token as `HF_TOKEN` |
| `servingruntime.yaml` | vLLM NVIDIA GPU ServingRuntime for KServe |
| `inference_huggingface.yaml` | InferenceService that pulls `hf://ibm-granite/granite-4.1-3b` |

You need a [Hugging Face access token](https://huggingface.co/settings/tokens) with permission to download `ibm-granite/granite-4.1-3b`. The cluster must reach Hugging Face on the network.

#### Update the hardware profile annotation

Edit `openshift-ai/model-hosting/inference_huggingface.yaml` and set `opendatahub.io/hardware-profile-name` to the hardware profile you created earlier.

#### From the command line

1. Log into the cluster with `oc login`.
2. Create the namespace:

```sh
oc apply -f openshift-ai/model-hosting/namespace.yml
```

3. Edit `openshift-ai/model-hosting/hf-secret.yaml`, replace `<your-hf-token>` with your token, then apply:

```sh
oc apply -f openshift-ai/model-hosting/hf-secret.yaml
```

4. Apply the ServingRuntime:

```sh
oc apply -f openshift-ai/model-hosting/servingruntime.yaml -n llm-serving
```

5. Apply the InferenceService:

```sh
oc apply -f openshift-ai/model-hosting/inference_huggingface.yaml
```

6. Wait until the deployment is ready:

```sh
oc get inferenceservice granite-41-3b -n llm-serving
oc get pods -n llm-serving -w
```

The InferenceService uses `storageUri: hf://ibm-granite/granite-4.1-3b` and injects `HF_TOKEN` from `hf-secret` so vLLM can authenticate to Hugging Face.

#### From the OpenShift AI UI

1. Log into the OpenShift AI dashboard with cluster-administrator (or project-admin) privileges.
2. Create a data science project named `llm-serving` (or select it if it exists).
3. Create a secret in `llm-serving` (OpenShift console **Workloads → Secrets**, or `oc`):
   - Name: `hf-secret`
   - Key: `HF_TOKEN`
   - Value: your Hugging Face access token
4. Ensure a vLLM NVIDIA GPU ServingRuntime is available. If not, apply `servingruntime.yaml` from the CLI or import it (**+ → Import YAML**).
5. Open the **Models** tab and click **Deploy model**.
6. Fill in the form:
   - **Model deployment name**: `granite-41-3b`
   - **Model type**: `Generative AI`
   - **Serving runtime**: `vLLM NVIDIA GPU ServingRuntime for KServe` (or `vllm-cuda-runtime`)
   - **Hardware profile**: match the profile you created (sample requests 1 GPU, 4 CPU, 32Gi–48Gi memory)
   - **Model location**: URI connection with:

     ```text
     hf://ibm-granite/granite-4.1-3b
     ```

7. Select **Make model deployment available through an external route**.
8. Enable **Require token authentication**.
9. Add an environment variable for Hugging Face auth:
   - Name: `HF_TOKEN`
   - Value from: secret `hf-secret`, key `HF_TOKEN`
10. Add custom runtime arguments, for example:
    - `--max-model-len=15000`
    - `--max-num-seqs=4`
    - `--gpu-memory-utilization=0.95`
    - `--enforce-eager`
    - `--enable-auto-tool-choice`
    - `--tool-call-parser=granite4`
11. Click **Deploy** and wait until the model is ready. The first pull from Hugging Face can take several minutes.

### Option B: Deploy from a container image or S3

Use this path only if you built a ModelCar image or uploaded the model to S3.

| File | Purpose |
|------|---------|
| `inference_ocimodel.yaml` | InferenceService using an OCI/ModelCar image |
| `inference_s3.yaml` | InferenceService using S3 storage |
| `s3-secret.yaml` | S3 credentials (required for S3 path) |
| `s3registry-sa.yaml` | ServiceAccount that mounts the S3 secret |

#### From the OpenShift AI UI

1. Open (or create) the `llm-serving` project.
2. Open the **Models** tab and click **Deploy model**.
3. Select the vLLM NVIDIA GPU ServingRuntime and configure replicas, size, and accelerator.
4. Set the model location:
   - **Container / OCI**: URI such as `oci://registry.example.com/models/granite:4.1-3b`
   - **S3**: connection to your bucket and path prefix (for example `granite413b`)
5. Configure serving arguments and authentication as needed, then click **Deploy**.

#### From the command line

1. Ensure the `llm-serving` namespace and ServingRuntime exist (same as Option A steps 2 and 4).
2. For S3: edit `s3-secret.yaml` with your credentials, then apply:

```sh
oc apply -f openshift-ai/model-hosting/s3-secret.yaml
oc apply -f openshift-ai/model-hosting/s3registry-sa.yaml
```

3. Edit `inference_ocimodel.yaml` or `inference_s3.yaml` for your storage URI, credentials, hardware profile, and resources.
4. Apply the InferenceService:

```sh
# OCI / ModelCar
oc apply -f openshift-ai/model-hosting/inference_ocimodel.yaml

# or S3
oc apply -f openshift-ai/model-hosting/inference_s3.yaml
```

5. Watch until the model is ready:

```sh
oc get pods -n llm-serving -w
```

### Test the model

Smoke-test the model before configuring LightSpeed:

```sh
POD=$(oc get pod -n llm-serving -o jsonpath='{.items[0].metadata.name}')

oc exec -n llm-serving "$POD" -c kserve-container -- \
  curl -s http://localhost:8080/v1/models | python3 -m json.tool
# Expect a model with id "granite-41-3b"

oc exec -n llm-serving "$POD" -c kserve-container -- \
  curl -s -X POST http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "granite-41-3b",
    "messages": [{"role": "user", "content": "What is a Kubernetes Deployment in one sentence?"}],
    "max_tokens": 100
  }' | python3 -m json.tool
```

If this fails, fix serving before configuring LightSpeed.

### Expose the model externally

> **NOTE:** This step is only required if you did not configure the inferenceService with `metadata.labels.networking.kserve.io/visibility: exposed`

List the predictor service:

```sh
oc get svc -n llm-serving
```

Example:

```text
NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
granite-41-3b-metrics     ClusterIP   172.30.165.244   <none>        8080/TCP   8m19s
granite-41-3b-predictor   ClusterIP   None             <none>        80/TCP     8m19s
```

Create a route (or edit `openshift-ai/model-hosting/route.yaml` and replace the sample host with your cluster domain):

```sh
oc create route edge --service=granite-41-3b-predictor -n llm-serving
# or
oc apply -f openshift-ai/model-hosting/route.yaml
```

Confirm the route:

```sh
$ oc get route -n llm-serving
NAME                      HOST/PORT                                                   PATH   SERVICES                  PORT   TERMINATION   WILDCARD
granite-41-3b-predictor   granite-41-3b-predictor-llm-serving.apps.sno.example.com    granite-41-3b-predictor          http   edge          None
```

Note the `HOST/PORT` value — you will use it as the LightSpeed provider URL.

---

## 4. Install OpenShift LightSpeed

Install LightSpeed on each cluster where you want the assistant available.

### From the OpenShift UI

1. Log into OpenShift with cluster-administrator privileges.
2. Go to **Ecosystem → Catalog**.
3. Search for **OpenShift LightSpeed**.
4. Select the latest version and click **Install**.

### From the command line

1. Apply the operator manifests:

```sh
oc apply -f lightspeed/operator
```

2. Wait for the operator to be ready:

```sh
oc get pods -n openshift-lightspeed
```

### Configure LightSpeed to use your model

1. Edit `lightspeed/olsconfig.yaml` and set the provider `url` to your model route, for example:

```text
https://granite-41-3b-predictor-llm-serving.apps.example.com/v1
```

Ensure `defaultModel` / model `name` match the served model id (`granite-41-3b`). Align `contextWindowSize` with the runtime `--max-model-len` (15000 in the Hugging Face sample; 35000 in the OCI sample).

2. If your InferenceService requires token authentication, put the token in `lightspeed/ols-secret.yaml` (`apitoken`). If auth is disabled on the model, a placeholder value is fine.

3. Apply the secret and config:

```sh
oc apply -f lightspeed/ols-secret.yaml
oc apply -f lightspeed/olsconfig.yaml
```

4. Confirm LightSpeed is running:

```sh
oc get pods -n openshift-lightspeed
```

You can then open the OpenShift console and use LightSpeed against your self-hosted Granite model.

#### Configure Lightspeed to trust self-signed certificate

Start by getting the certificate in question:

```
echo | openssl s_client -connect example.com:443 2>&1 | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > server-cert.pem
```

Now create a configMap from the file above:

```
oc create configmap trusted-certs --from-file=server-cert.pem --namespace openshift-lightspeed
```

Finally update the OLS configuration to include the configmap 

```
apiVersion: ols.openshift.io/v1alpha1
kind: OLSConfig
metadata:
  name: cluster
spec:
  ols:
    additionalCAConfigMapRef:
      name: trusted-certs
```

# References

[Lightspeed Documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_lightspeed/1.0)
[Configuring LightSpeed with Trusted Certs](https://docs.redhat.com/en/documentation/red_hat_openshift_lightspeed/1.0/html/configure/ols-configuring-openshift-lightspeed#ols-support-for-trusted-ca-certificates-and-llm-providers_ols-configuring-openshift-lightspeed)