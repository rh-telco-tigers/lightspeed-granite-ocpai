# Knowledge Base Web Server

Simple Markdown HTTP server for the BYOK lab knowledge base. It renders the files under `byok/knowledgebase/` so LightSpeed citations can link back to readable documentation.

Uses [markdownd](https://github.com/aerth/markdownd).

## Build and push to a registry

Run from the `byok/` directory so the build context includes `knowledgebase/`:

```sh
cd byok
podman build -f kbserver/Containerfile -t registry.example.com/kbserver/kbserver:latest .
podman push registry.example.com/kbserver/kbserver:latest
```

## Deploy to OpenShift

Before running the commands below, update `spec.template.spec.containers.image` in `kbserver/deployment/01_kbserver-deployment.yaml` to point to your registry image.

```sh
oc login
oc apply -f byok/kbserver/deployment
oc get route -n kbserver
```

Example output:

```text
NAME       HOST/PORT                                 PATH   SERVICES   PORT   TERMINATION     WILDCARD
kbserver   kbserver-kbserver.apps.sno.example.com           kbserver   8080   edge/Redirect   None
```

## URL alignment with knowledge base metadata

Before building the BYOK RAG image, update the `url` fields in the knowledge base files to match the Route hostname from your deployment:

```sh
cd byok/knowledgebase
sed -i 's/docs.granitelab.example.com/kbserver-kbserver.apps.sno.example.com/g' *.md
```

After updating the URLs, rebuild the BYOK RAG image (see the main BYOK lab, section 2).

## Run locally

```sh
cd byok
podman build -f kbserver/Containerfile -t kbserver:latest .
podman run --rm -p 8080:8080 kbserver:latest
```

Open `http://localhost:8080/` for the generated index, or request a file directly (for example `http://localhost:8080/deployment-standards.md`).
