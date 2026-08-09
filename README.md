# Story Generator for Feuerwehr Emmerich Instagram

Story Generator for the @feuerwehremmerich Instagram account. Make generating stories for incidents easy and consistent!

Thanks to Floris Wichert (@loeschwerk) for providing the HTML file.

## Architecture

The architecture is simple - in the `site/` directory, all files can be hosted directly on a webserver. This repo adds support for Docker and Kubernetes to deploy on infrastructure.

## Production deployment

The production workload is defined by the root `kustomization.yaml` and the
manifests in `k8s/`. Configure the Argo CD application to use this repository,
track the `main` branch, and use `.` as its path. Enable
automatic sync if each successful workflow run should be rolled out without a
manual approval in Argo CD.

The GitHub Actions workflow validates the Kustomize deployment on every push to
`main`. Argo CD detects the same push and deploys the manifests. The site and
nginx configuration are mounted into the stock `nginx:1.31-alpine` image as
ConfigMaps, so this project does not need to build or publish a container image.

Before the first Argo CD sync, create the Basic Auth Secret manually:

```sh
kubectl apply -f k8s/namespace.yaml
htpasswd -cB .htpasswd YOUR_USERNAME
kubectl -n incident-story-generator create secret generic nginx-basic-auth \
  --from-file=.htpasswd=.htpasswd
rm .htpasswd
```

For later credential changes, recreate the local file and use `kubectl create
secret ... --dry-run=client -o yaml | kubectl apply -f -`. The `.htpasswd` file
and credentials must not be committed. Argo CD owns the Deployment, Service,
and ConfigMaps; you own the sensitive Secret separately.

## Domain and HTTPS

Set `DNS_NAME` in `k8s/domain-config.yaml`. Kustomize copies it into the Traefik
Ingress and Certificate. Argo CD creates both resources using the cluster's
existing `letsencrypt-prod` ClusterIssuer. cert-manager completes the HTTP-01 challenge,
stores the issued certificate in `incident-story-generator-tls`, and renews it
automatically. Traefik reads that Secret through the Ingress TLS configuration.

The cluster must have cert-manager and the `letsencrypt-prod` ClusterIssuer
installed. No Traefik
certificate resolver and no manually created TLS Secret are needed. The
domain's A/AAAA records must point to Traefik, and its HTTP entry point must be
publicly reachable so Let's Encrypt can complete the HTTP-01 challenge.
