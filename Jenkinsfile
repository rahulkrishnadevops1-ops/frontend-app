pipeline {
  agent none

  options {
    skipDefaultCheckout()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  parameters {
    string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch to build')
    string(name: 'IMAGE_REPOSITORY', defaultValue: 'privatergistry/nginx-demo', description: 'Container image repository')
    string(name: 'IMAGE_TAG', defaultValue: '', description: 'Image tag override (leave empty to use BUILD_NUMBER)')
    string(name: 'RELEASE_NAME', defaultValue: 'nginx-demo', description: 'Helm release name')
    string(name: 'K8S_NAMESPACE', defaultValue: 'jenkins', description: 'Kubernetes namespace to deploy into')
    string(name: 'HELM_CHART_PATH', defaultValue: 'helm/nginx-demo', description: 'Path to Helm chart')
  }

  environment {
    REGISTRY_CREDENTIALS_ID = 'dockerhub-creds'
    EFFECTIVE_TAG = ''
    RESOLVED_IMAGE_REPOSITORY = ''
  }

  stages {
    stage('Checkout') {
      agent any
      steps {
        git branch: "${params.BRANCH}", url: 'https://github.com/Narendra-Geddam/nginx-demo.git'
        script {
          def requestedTag = params.IMAGE_TAG?.trim()
          env.EFFECTIVE_TAG = (requestedTag && requestedTag != 'null') ? requestedTag : (env.BUILD_NUMBER ?: 'latest')
        }
        stash name: 'source', includes: '**/*'
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
        unstash 'source'
        container('kaniko') {
          withCredentials([usernamePassword(credentialsId: env.REGISTRY_CREDENTIALS_ID, usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
            script {
              def imageRepo = params.IMAGE_REPOSITORY?.trim()
              if (!imageRepo || imageRepo == 'null') {
                imageRepo = "${REG_USER}/nginx-demo"
              }
              if (!imageRepo.contains('/')) {
                error("Invalid IMAGE_REPOSITORY '${imageRepo}'. Use '<namespace>/<repo>' format.")
              }
              def registryHost = imageRepo.tokenize('/')[0]
              def isDockerHub = !(registryHost.contains('.') || registryHost.contains(':') || registryHost == 'localhost')
              if (isDockerHub && !imageRepo.startsWith("${REG_USER}/")) {
                def repoName = imageRepo.split('/', 2)[1]
                echo "Adjusting IMAGE_REPOSITORY to Docker Hub credential namespace: ${REG_USER}/${repoName}"
                imageRepo = "${REG_USER}/${repoName}"
              }
              def effectiveTag = env.EFFECTIVE_TAG?.trim()
              if (!effectiveTag || effectiveTag == 'null') {
                effectiveTag = env.BUILD_NUMBER ?: 'latest'
              }
              env.RESOLVED_IMAGE_REPOSITORY = imageRepo
              env.EFFECTIVE_TAG = effectiveTag
              withEnv([
                "IMAGE_REPOSITORY=${imageRepo}",
                "EFFECTIVE_TAG=${effectiveTag}"
              ]) {
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
        unstash 'source'
        container('helm') {
          script {
            def imageRepo = env.RESOLVED_IMAGE_REPOSITORY?.trim() ? env.RESOLVED_IMAGE_REPOSITORY.trim() : params.IMAGE_REPOSITORY?.trim()
            def releaseName = params.RELEASE_NAME?.trim()
            def namespace = params.K8S_NAMESPACE?.trim()
            def chartPath = params.HELM_CHART_PATH?.trim()
            def effectiveTag = env.EFFECTIVE_TAG?.trim() ? env.EFFECTIVE_TAG.trim() : (env.BUILD_NUMBER ?: 'latest')
            withEnv([
              "IMAGE_REPOSITORY=${imageRepo}",
              "RELEASE_NAME=${releaseName}",
              "K8S_NAMESPACE=${namespace}",
              "HELM_CHART_PATH=${chartPath}",
              "EFFECTIVE_TAG=${effectiveTag}"
            ]) {
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
      }
    }

    stage('Post-Deploy Verify') {
      agent any
      steps {
        script {
          withEnv([
            "RELEASE_NAME=${params.RELEASE_NAME?.trim()}",
            "K8S_NAMESPACE=${params.K8S_NAMESPACE?.trim()}"
          ]) {
            sh '''
              kubectl get deploy,po,svc,ingress -n "${K8S_NAMESPACE}" -l app.kubernetes.io/instance="${RELEASE_NAME}"
            '''
          }
        }
      }
    }
  }
}
