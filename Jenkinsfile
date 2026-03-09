pipeline {
  agent none

  parameters {
    string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch')
    string(name: 'IMAGE_REPOSITORY', defaultValue: 'privatergistry/nginx-demo', description: 'Docker image repo (ex: user/nginx-demo)')
    string(name: 'IMAGE_TAG', defaultValue: '', description: 'Leave empty to use BUILD_NUMBER')
    string(name: 'RELEASE_NAME', defaultValue: 'nginx-demo', description: 'Helm release')
    string(name: 'K8S_NAMESPACE', defaultValue: 'jenkins', description: 'K8s namespace')
    string(name: 'HELM_CHART_PATH', defaultValue: 'helm/nginx-demo', description: 'Helm chart path')
  }

  environment {
    REGISTRY_CREDENTIALS_ID = 'dockerhub-creds'
    FINAL_IMAGE_REPOSITORY = ''
    FINAL_IMAGE_TAG = ''
  }

  stages {
    stage('Build and Push Image') {
      agent {
        kubernetes {
          yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.23.2-debug
    command: ["/busybox/cat"]
    tty: true
"""
        }
      }
      steps {
        git branch: "${params.BRANCH}", url: 'https://github.com/Narendra-Geddam/nginx-demo.git'
        container('kaniko') {
          withCredentials([usernamePassword(credentialsId: env.REGISTRY_CREDENTIALS_ID, usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
            script {
              def tag = params.IMAGE_TAG?.trim() ? params.IMAGE_TAG.trim() : env.BUILD_NUMBER
              def imageRepo = params.IMAGE_REPOSITORY?.trim()
              if (imageRepo && imageRepo.contains('/') && !imageRepo.contains('.') && !imageRepo.contains(':')) {
                def repoName = imageRepo.split('/', 2)[1]
                imageRepo = "${REG_USER}/${repoName}"
              }
              env.FINAL_IMAGE_REPOSITORY = imageRepo
              env.FINAL_IMAGE_TAG = tag
              withEnv(["IMAGE_REPOSITORY=${imageRepo}", "IMAGE_TAG=${tag}"]) {
                sh '''
                  mkdir -p /kaniko/.docker
                  AUTH=$(printf "%s:%s" "$REG_USER" "$REG_PASS" | base64 | tr -d '\n')
                  cat > /kaniko/.docker/config.json <<EOF
                  {"auths":{"https://index.docker.io/v1/":{"auth":"$AUTH"}}}
EOF
                  /kaniko/executor \
                    --context "$PWD" \
                    --dockerfile "$PWD/Dockerfile" \
                    --destination "${IMAGE_REPOSITORY}:${IMAGE_TAG}" \
                    --destination "${IMAGE_REPOSITORY}:latest"
                '''
              }
            }
          }
        }
      }
    }

    stage('Deploy with Helm') {
      agent {
        kubernetes {
          yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
  - name: helm
    image: alpine/helm:3.14.4
    command: ["/bin/sh","-c","cat"]
    tty: true
"""
        }
      }
      steps {
        git branch: "${params.BRANCH}", url: 'https://github.com/Narendra-Geddam/nginx-demo.git'
        container('helm') {
          script {
            def imageRepo = env.FINAL_IMAGE_REPOSITORY?.trim() ? env.FINAL_IMAGE_REPOSITORY : params.IMAGE_REPOSITORY
            def tag = env.FINAL_IMAGE_TAG?.trim() ? env.FINAL_IMAGE_TAG : (params.IMAGE_TAG?.trim() ? params.IMAGE_TAG.trim() : env.BUILD_NUMBER)
            withEnv(["IMAGE_REPOSITORY=${imageRepo}", "IMAGE_TAG=${tag}"]) {
              sh '''
                helm upgrade --install "${RELEASE_NAME}" "${HELM_CHART_PATH}" \
                  --namespace "${K8S_NAMESPACE}" \
                  --set image.repository="${IMAGE_REPOSITORY}" \
                  --set image.tag="${IMAGE_TAG}" \
                  --wait --timeout 5m
              '''
            }
          }
        }
      }
    }
  }
}
