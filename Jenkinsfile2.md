# Jenkinsfile2 Pipeline Guide

This document is only for `Jenkinsfile2`.

## Purpose

`Jenkinsfile2` runs on Jenkins controller (`built-in`) and does:
1. Checkout source.
2. Validate required tools (`kubectl`, `helm`, `docker`).
3. Build Docker image.
4. Push image to Docker Hub.
5. Deploy to Minikube using Helm and kubeconfig secret file.

## Required Jenkins Credentials

1. `dockerhub-creds`
   - Type: `Username with password`
   - Used for `docker login` and push.
2. `minikube-kubeconfig`
   - Type: `Secret file`
   - Exported as `KUBECONFIG` for `kubectl` and `helm`.

## Jenkinsfile2 Parameters

1. `KUBECONFIG_CREDENTIALS_ID` (default: `minikube-kubeconfig`)
2. `DOCKER_CREDENTIALS_ID` (default: `dockerhub-creds`)
3. `IMAGE_REPOSITORY` (default: `privatergistry/nginx-demo`)
4. `IMAGE_TAG` (default: empty; uses `BUILD_NUMBER`)
5. `RELEASE_NAME` (default: `nginx-demo`)
6. `K8S_NAMESPACE` (default: `dev`)
7. `HELM_CHART_PATH` (default: `helm/nginx-demo`)
8. `VALUES_FILE` (default: `helm/nginx-demo/values.yaml`)

## Deploy Command Logic

Helm deploy step uses:
- `helm upgrade --install`
- `--create-namespace`
- `--wait --atomic --timeout 5m`
- image overrides:
  - `--set image.repository=<resolved repository>`
  - `--set image.tag=<resolved tag>`

Then rollout verification runs:

```bash
kubectl rollout status deployment/<RELEASE_NAME> -n <K8S_NAMESPACE> --timeout=180s
```

## Troubleshooting

1. `No such DSL method withKubeConfig`
   - `Jenkinsfile2` does not require that step.
   - Ensure credential is `Secret file` and ID matches `KUBECONFIG_CREDENTIALS_ID`.
2. `ImagePullBackOff`
   - Confirm pushed image exists in Docker Hub.
   - Confirm repository/tag in build logs.
3. `docker login` failure
   - Recheck `dockerhub-creds` username/password.

