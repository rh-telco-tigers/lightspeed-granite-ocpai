# Bring Your Own Knowledge (BYOK) Lab

This lab extends the main tutorial. After you host Granite 4.2-3B and connect OpenShift LightSpeed, you can augment LightSpeed with **institutional knowledge** using the Bring Your Own Knowledge (BYOK) feature.

BYOK ingests Markdown documentation, builds a vector index, packages it as a container image, and configures LightSpeed to retrieve from that index during chat.

> **Technology Preview:** BYOK and the RAG build tool are Technology Preview features. See the [Red Hat documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_lightspeed/1.0/html/configure/ols-configuring-openshift-lightspeed#providing-custom-knowledge-to-the-llm_ols-configuring-openshift-lightspeed) for support scope.

## Prerequisites

Complete the main lab first:

- OpenShift LightSpeed installed and running
- `OLSConfig` configured with your self-hosted Granite model (see `lightspeed/olsconfig.yaml`)
- `oc` and `podman` on a workstation or RHEL 9 host with network access to your cluster and `registry.redhat.io`
- FUSE overlay support (for example, `fuse-overlayfs`):

```sh
sudo dnf install fuse-overlayfs
```

## Sample knowledge base

This repo includes a fictional **Granite Lab Industries** standards library in `byok/knowledgebase/`. All files are Markdown (`.md`), which is required by the RAG tool.

| File | Topic |
|------|--------|
| `deployment-standards.md` | Replicas, SCCs, resources, probes |
| `global-metadata-standards.md` | Required labels and annotations |
| `network-policy-standards.md` | Zero-trust network policies |
| `enterprise-network-connectivity.md` | VLAN topology, firewall rules, OpenShift-to-database connectivity |
| `configmap-secret-standards.md` | ConfigMap and Secret conventions |

You can replace these files with your own organizational docs. Optional YAML front matter sets document title and URL metadata:

```yaml
---
title: "My Document Title"
url: "https://docs.example.com/my-doc"
---
```

If front matter is omitted, the first `#` heading becomes the title and the file path becomes the URL.

### Extra credit

OpenShift LightSpeed shows the source of articles it cites based on the `url` field in each Markdown file's front matter. To demo clickable source links, follow the instructions in [`byok/kbserver`](kbserver/README.md) to deploy a simple Markdown web server for the sample knowledge base.

---

## 1. Configure environment variables

Copy the example environment file and edit it for your cluster:

```sh
cp byok/env.example byok/env.local
```

| Variable | Description |
|----------|-------------|
| `MARKDOWN_SOURCE_DIR` | Path to Markdown knowledge files |
| `OUTPUT_DIR` | Where the RAG tool writes `byok-image.tar` |
| `BYOK_IMAGE` | Full image reference to push and use in OLSConfig |
| `OCP_REGISTRY_URL` | OpenShift internal registry hostname |
| `OLS_NAMESPACE` | LightSpeed namespace (default `openshift-lightspeed`) |
| `RAG_INDEX_ID` | Vector index id (default `vector_db_index`) |
| `RAG_INDEX_PATH` | Path inside the image (default `/rag/vector_db`) |

Load the file before running the commands in the following sections:

```sh
source byok/env.local
mkdir -p "${OUTPUT_DIR}"
```

---

## 2. Build the RAG image

1. Log in to the Red Hat registry (required to pull the RAG tool image):

```sh
podman login registry.redhat.io
```

2. Run the RAG tool to build the vector index and image archive:

```sh
podman run -it --rm --device=/dev/fuse \
  -v "$XDG_RUNTIME_DIR/containers/auth.json:/run/user/0/containers/auth.json:Z" \
  -v "$(pwd)/${MARKDOWN_SOURCE_DIR}:/markdown:Z" \
  -v "$(pwd)/${OUTPUT_DIR}:/output:Z" \
  registry.redhat.io/openshift-lightspeed-tech-preview/lightspeed-rag-tool-rhel9:latest
```

This reads Markdown from `byok/knowledgebase/` and writes `byok/output/byok-image.tar`.

### Troubleshooting the RAG build

If you see `Error: 'overlay' is not supported over overlayfs, a mount_program is required: backing file system is unsupported for this graph driver`, update your Podman storage configuration.

Edit one of the following files:

* Rootless users: `~/.config/containers/storage.conf`
* Root or system-wide: `/etc/containers/storage.conf`

Add the following for overlay with FUSE:

```ini
[storage]
driver = "overlay"

[storage.options.overlay]
mount_program = "/usr/bin/fuse-overlayfs"
force_mask = "shared"
```

Or, if you use a Btrfs file system:

```ini
[storage]
driver = "btrfs"
```

Then reset Podman storage with `podman system reset`. This removes all local images and containers, so stop running pods first and plan to re-pull images afterward.


---

## 3. Push the image to a registry

1. Set the image reference in `byok/env.local`, for example:

```text
BYOK_IMAGE="quay.io/<username>/byok-image:latest"
```

2. Log in, load, tag, and push:

```sh
source byok/env.local
podman login quay.io
podman load -i "${OUTPUT_DIR}/byok-image.tar"
podman tag localhost/byok-image:latest "${BYOK_IMAGE}"
podman push "${BYOK_IMAGE}"
```

If the registry is private, create an image pull secret in `openshift-lightspeed` and add it to `spec.ols.imagePullSecrets` in OLSConfig.

---

## 4. Configure LightSpeed for BYOK

Patch the `cluster` OLSConfig to reference your RAG image:

```sh
source byok/env.local

oc patch olsconfig cluster --type merge -p "{
  \"spec\": {
    \"ols\": {
      \"rag\": [
        {
          \"image\": \"${BYOK_IMAGE}\",
          \"indexID\": \"${RAG_INDEX_ID}\",
          \"indexPath\": \"${RAG_INDEX_PATH}\"
        }
      ]
    }
  }
}"
```

This adds configuration like:

```yaml
spec:
  ols:
    rag:
      - image: <your-registry>/openshift-lightspeed/byok-image:latest
        indexID: vector_db_index
        indexPath: /rag/vector_db
```

You can also edit OLSConfig manually in the console (**Operators → Installed Operators → OpenShift LightSpeed → OLSConfig → YAML**) using `byok/olsconfig-rag-snippet.yaml` as a reference.

Wait for LightSpeed pods to reconcile:

```sh
oc get pods -n openshift-lightspeed -w
```

### Optional: use only custom knowledge

To disable the built-in OpenShift documentation RAG sidecar and answer only from your BYOK index, set:

```yaml
spec:
  ols:
    byokRAGOnly: true
```

---

## 5. Test BYOK in the console

Open the OpenShift console LightSpeed chat and try questions grounded in the sample knowledge base:

- *What security context settings does Granite Lab Industries require for containers?*
- *What labels must Granite Lab deployments include?*
- *Tell me about Granite Lab network policy standards.*
- *Why can't my OpenShift app on VLAN25 connect to the PostgreSQL database on VLAN15?*
- *What firewall rules allow OpenShift egress to reach databases on 172.16.15.0/24?*
- *My app times out connecting to db-oracle-fin-01 — what should I check?*

Responses should reference Granite Lab standards and network documentation instead of generic OpenShift guidance alone.

---

## 6. Cleanup

Remove the BYOK RAG configuration:

```sh
oc patch olsconfig cluster --type json \
  -p='[{"op": "remove", "path": "/spec/ols/rag"}]' 2>/dev/null || \
oc patch olsconfig cluster --type merge -p '{"spec":{"ols":{"rag":[]}}}'
```

To rebuild after editing Markdown files, rerun steps 2–4. LightSpeed can detect updates when using floating tags such as `latest`.

---

## References

- [Providing custom knowledge to the LLM](https://docs.redhat.com/en/documentation/red_hat_openshift_lightspeed/1.0/html/configure/ols-configuring-openshift-lightspeed#providing-custom-knowledge-to-the-llm_ols-configuring-openshift-lightspeed)
- [OpenShift LightSpeed — BYOK Demo](https://github.com/xphyr/LightspeedBYOKDemo) (extended demo with remote RHEL workflow)

## Acknowledgements

> **AI-assisted authoring**
>
> The lab design, topic, concept, and workflow are original to the author. Formatting, layout, and readability improvements were assisted by AI tools (including Grok 4.5).
