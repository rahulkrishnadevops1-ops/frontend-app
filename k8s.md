# NGINX Demo on Kubernetes with Jenkins CI/CD

Minimal lab setup (cleaned):
1. Jenkins runs in Kubernetes.
2. Jenkins controller builds and pushes image to Docker Hub.
3. Helm deploys app.
4. App is exposed with Service `NodePort` on port `30081`.

---

## Repository Layout

```text
.
├── Dockerfile
├── index.html
├── Jenkinsfile
├── Jenkinsfile2
├── helm/nginx-demo/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── _helpers.tpl
│       ├── deployment.yaml
│       └── service.yaml
└── k8s/
    ├── jenkins-values.yaml
    ├── jenkins-serviceaccount.yaml
    ├── jenkins-deployment.yaml
    └── jenkins-helm-cluster-rbac.yaml
```

---

## 1) Install Local Path Provisioner and Set Default StorageClass

Install local-path provisioner:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

Mark `local-path` as default StorageClass:

```bash
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Verify:

```bash
kubectl get storageclass
```

You should see `local-path` with `(default)`.

---

## 2) Install Jenkins

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update

kubectl create namespace jenkins --dry-run=client -o yaml | kubectl apply -f -
```

### Option A: Install with local values file (recommended)

Use repo file: `k8s/jenkins-values.yaml`

```bash
helm upgrade --install jenkins jenkins/jenkins \
  -n jenkins \
  -f k8s/jenkins-values.yaml
```

This values file sets:
- Jenkins Service type `NodePort`
- Jenkins NodePort HTTP `32000`
- Jenkins ingress enabled with host `jenkins.local` (class `nginx`)
- Persistence enabled with `local-path` StorageClass and `10Gi`

If you are already inside folder containing this file, the command is:

```bash
helm upgrade --install jenkins jenkins/jenkins \
  -n jenkins \
  -f values.yaml
```

### Option B: Inline set (quick)

```bash
helm upgrade --install jenkins jenkins/jenkins \
  -n jenkins \
  --set controller.serviceType=NodePort
```

Verify:

```bash
kubectl get pods -n jenkins
kubectl get svc -n jenkins
kubectl get ingress -n jenkins
```

---

## 3) Apply Jenkins RBAC

```bash
kubectl apply -f k8s/jenkins-serviceaccount.yaml
kubectl apply -f k8s/jenkins-helm-cluster-rbac.yaml
```

Verify:

```bash
kubectl auth can-i list secrets --as=system:serviceaccount:jenkins:jenkins -n dev
```

Expected: `yes`

---

## 4) Jenkins Pipeline Parameters

- `KUBECONFIG_CREDENTIALS_ID` (default: `minikube-kubeconfig`)
- `DOCKER_CREDENTIALS_ID` (default: `dockerhub-creds`)
- `IMAGE_REPOSITORY` (default: `privatergistry/nginx-demo`)
- `IMAGE_TAG` (empty = `BUILD_NUMBER`)
- `RELEASE_NAME` (default: `nginx-demo`)
- `K8S_NAMESPACE` (default: `dev`)
- `HELM_CHART_PATH` (default: `helm/nginx-demo`)
- `VALUES_FILE` (default: `helm/nginx-demo/values.yaml`)

Jenkinsfile used for this flow: `Jenkinsfile2`

Required Jenkins credentials:
- `dockerhub-creds` of type `Username with password`
- `minikube-kubeconfig` of type `Secret file`

Detailed pipeline reference:
- `Jenkinsfile2.md`

---

## 5) Current App Exposure

Helm chart defaults:
- `service.type: NodePort`
- `service.port: 80`
- `service.targetPort: 8080`
- `service.nodePort: 30081`

No ingress resource is created in this cleaned lab setup.

---

## 6) Validate Deployment

```bash
kubectl get deploy,po,svc -n dev
helm list -n dev
kubectl get deployment nginx-demo -n dev -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
```

Access app:

```text
http://<NODE_IP>:30081
```

---

## 7) Troubleshooting

### Pod fails with permission errors on NGINX temp/cache dirs
Use the current Dockerfile base image:
- `nginxinc/nginx-unprivileged:stable-alpine`

### `secrets is forbidden` or `serviceaccounts is forbidden`
Re-apply:
- `k8s/jenkins-helm-cluster-rbac.yaml`

### `No such DSL method withKubeConfig`
Your Jenkins does not have that plugin step.
Use `Jenkinsfile2`, which binds kubeconfig through `withCredentials([file(...)])`.

### Jenkins UI Access
- Direct NodePort: `http://<NODE_IP>:32000`
- Ingress (if DNS/hosts configured): `http://jenkins.local`

---

## 8) Quick Checklist

1. `dockerhub-creds` exists in Jenkins.
2. `minikube-kubeconfig` exists as `Secret file`.
3. Jenkins SA exists in `jenkins` namespace.
4. `local-path` StorageClass is default.
5. Cluster RBAC applied.
6. Pipeline (`Jenkinsfile2`) completed successfully.
7. `kubectl get svc -n dev` shows `NodePort` `30081`.
