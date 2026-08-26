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

Before generating the BYOK container file, update the files in the `knowledgebase` directory with the route that is created:

```sh
cd knowledgebase
sed -i s/docs.granitelab.example.com/kbserver-kbserver.apps.sno.xphyrlab.net/g
```
