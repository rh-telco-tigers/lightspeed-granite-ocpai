# Knowledge Base Web Server

Simple Markdown HTTP server for the BYOK lab knowledge base. It renders the files under `byok/knowledgebase/` so LightSpeed citations can link back to readable documentation.

Uses [markdownd](https://github.com/aerth/markdownd).

## Build

Run from the `byok/` directory so the build context includes `knowledgebase/`:

```sh
cd byok
podman build -f kbserver/Containerfile -t kbserver:latest .
```

## Run locally

```sh
podman run --rm -p 8080:8080 kbserver:latest
```

Open `http://localhost:8080/` for the generated index, or request a file directly (for example `http://localhost:8080/deployment-standards.md`).

## URL alignment with knowledge base metadata

Knowledge articles use URLs such as `https://docs.granitelab.example.com/openshift/deployment-standards.html`. This image serves files from `/opt` at paths like `/deployment-standards.md`. To match the metadata URLs in a lab, point an OpenShift Route or ingress at this service and add path-based routing, or update the `url` fields in the Markdown front matter to match how you expose the service.
