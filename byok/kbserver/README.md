# Knowledge Base Web Server

Simple Markdown HTTP server for the BYOK lab knowledge base. It renders the files under `byok/knowledgebase/` so LightSpeed citations can link back to readable documentation.

Uses [markdownd](https://github.com/aerth/markdownd).

## Build and Push to Registry

Run from the `byok/` directory so the build context includes `knowledgebase/`:

```sh
cd byok
podman build -f kbserver/Containerfile -t registry.example.com/kbserver/kbserver:latest .
podman push registry.example.com/kbserver/kbserver:latest
```

## Deploy to OpenShift

Before running the following comamnds, be sure to update `spec.template.spec.Containers.image` path to point to your registry.

```sh
oc login
oc apply -f byok/kbserver/deployment
oc get route -n kbserver
```

## URL alignment with knowledge base metadata

Knowledge articles use URLs such as `https://docs.granitelab.example.com/openshift/deployment-standards.html`. This image serves files from `/opt` at paths like `/deployment-standards.md`. To match the metadata URLs in a lab, point an OpenShift Route or ingress at this service and add path-based routing, or update the `url` fields in the Markdown front matter to match how you expose the service.
