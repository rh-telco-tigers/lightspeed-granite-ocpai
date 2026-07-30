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
2. run the following command `oc apply -f openshift-ai\operator`
> Note: if you see any failures re-apply the missed files

#### 


## Building the Granite41 model container


There are multiple ways to get a copy of the Granite41 model, for hosting. We will use the registry hosting model to do this. To start we will need to have a copy of the `huggingface` utilites on your machine. See [Install the Huggingface CLI](https://huggingface.co/docs/huggingface_hub/main/en/installation#install-the-hugging-face-cli) for details.

Start by downloading the model we will be using:

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

Now that we have installed OpenShift AI and built our Granite41 model container we can start hosting the model

### Hosting the Model via the UI


### Hosting the Model via the command line


## Installing OpenShift LightSpeed

On the cluster(s) you want to run OpenShift LightSpeed on

To install OpenShift LightSpeed from the UI, follow these steps:
1. log into OpenShift with cluster administrator privledges
2. Select "Ecosystem"->"Catalog"
3. Search for "OpenShift LightSpeed"
4. Select Install
5. Select the latest version
6. Click Install