pipeline {
  agent none

  options {
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  parameters {
    string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch to build')
    string(name: 'IMAGE_REPOSITORY', defaultValue: 'privatergestry/nginx-demo', description: 'Container image repository')
    string(name: 'IMAGE_TAG', defaultValue: '', description: 'Image tag override (leave empty to use BUILD_NUMBER)')
    string(name: 'RELEASE_NAME', defaultValue: 'nginx-demo', description: 'Helm release name')
    string(name: 'K8S_NAMESPACE', defaultValue: 'jenkins', description: 'Kubernetes namespace to deploy into')
    string(name: 'HELM_CHART_PATH', defaultValue: 'helm/nginx-demo', description: 'Path to Helm chart')
  }

  environment {
    REGISTRY_CREDENTIALS_ID = 'dockerhub-creds'
    EFFECTIVE_TAG = ''
  }

  stages {
    stage('Checkout') {
      agent any
      steps {
        git branch: "${params.BRANCH}", url: 'https://github.com/Narendra-Geddam/nginx-demo.git'
        script {
          env.EFFECTIVE_TAG = params.IMAGE_TAG?.trim() ? params.IMAGE_TAG.trim() : env.BUILD_NUMBER
        }
      }
    }

    stage('Build and Push (Kaniko)') {
      agent {
        kubernetes {
          yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: kaniko-builder
spec:
  serviceAccountName: jenkins
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.23.2-debug
    command:
    - /busybox/cat
    tty: true
"""
        }
      }
      steps {
        container('kaniko') {
          withCredentials([usernamePassword(credentialsId: env.REGISTRY_CREDENTIALS_ID, usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
            sh '''
              REGISTRY_HOST=$(echo "${IMAGE_REPOSITORY}" | cut -d/ -f1)
              if ! echo "$REGISTRY_HOST" | grep -Eq '[.:]|localhost'; then
                REGISTRY_HOST='https://index.docker.io/v1/'
              fi
              mkdir -p /kaniko/.docker
              AUTH=$(printf "%s:%s" "$REG_USER" "$REG_PASS" | base64 | tr -d '\n')
              cat > /kaniko/.docker/config.json <<EOF
              {
                "auths": {
                  "$REGISTRY_HOST": { "auth": "$AUTH" }
                }
              }
EOF

              /kaniko/executor \
                --context "$PWD" \
                --dockerfile "$PWD/Dockerfile" \
                --destination "${IMAGE_REPOSITORY}:${EFFECTIVE_TAG}" \
                --destination "${IMAGE_REPOSITORY}:latest"
            '''
          }
        }
      }
    }

    stage('Helm Deploy') {
      agent {
        kubernetes {
          yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: helm-deployer
spec:
  serviceAccountName: jenkins
  containers:
  - name: helm
    image: alpine/helm:3.14.4
    command:
    - /bin/sh
    - -c
    - cat
    tty: true
"""
        }
      }
      steps {
        container('helm') {
          sh '''
            helm lint "${HELM_CHART_PATH}"
            helm upgrade --install "${RELEASE_NAME}" "${HELM_CHART_PATH}" \
              --namespace "${K8S_NAMESPACE}" \
              --create-namespace \
              --wait --timeout 5m \
              --set image.repository="${IMAGE_REPOSITORY}" \
              --set image.tag="${EFFECTIVE_TAG}" \
              --set image.pullPolicy="Always"
          '''
        }
      }
    }

    stage('Post-Deploy Verify') {
      agent any
      steps {
        sh '''
          kubectl get deploy,po,svc,ingress -n "${K8S_NAMESPACE}" -l app.kubernetes.io/instance="${RELEASE_NAME}"
        '''
      }
    }
  }
}
