pipeline {
    agent { label 'worker-node' }
 
    environment {
        APP_NAME           = 'my-app'
        IMAGE_TAG          = "${BUILD_NUMBER}"
        CONTAINER_PORT     = '80'
        HOST_PORT          = '80'
        DOCKERHUB_USERNAME = credentials('DOCKERHUB_USERNAME')
    }
 
    triggers {
        githubPush()
    }
 
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }
 
    stages {
 
        stage('Checkout') {
            steps {
                checkout scm
                echo "Branch: ${env.BRANCH_NAME} | Build: #${BUILD_NUMBER} | Commit: ${GIT_COMMIT[0..7]}"
            }
        }
 
        stage('Code Quality') {
            steps {
                sh '''
                  echo "=== HTML Validation ==="
                  if [ -f index.html ]; then
                    python3 -c "
import html.parser
class V(html.parser.HTMLParser):
    def __init__(self): super().__init__()
v=V()
v.feed(open('index.html').read())
print('HTML syntax OK')
"
                  fi
                  echo "=== File checks ==="
                  test -f index.html && echo "index.html exists" || exit 1
                  test -f Dockerfile && echo "Dockerfile exists" || exit 1
                '''
            }
        }
 
        stage('Security Scan - Source') {
            steps {
                sh '''
                  trivy fs --exit-code 0 \
                    --severity HIGH,CRITICAL \
                    --format table \
                    --output trivy-fs-report.txt .
                  cat trivy-fs-report.txt
                '''
            }
            post {
                always { archiveArtifacts artifacts: 'trivy-fs-report.txt', fingerprint: true }
            }
        }
 
        stage('Build Docker Image') {
            steps {
                sh '''
                  export DOCKER_IMAGE=${DOCKERHUB_USERNAME}/${APP_NAME}
                  echo "Building ${DOCKER_IMAGE}:${IMAGE_TAG}"
                  docker build \
                    --label "build=${BUILD_NUMBER}" \
                    --label "branch=${BRANCH_NAME}" \
                    --label "git-commit=${GIT_COMMIT}" \
                    -t ${DOCKER_IMAGE}:${IMAGE_TAG} \
                    -t ${DOCKER_IMAGE}:latest .
                '''
            }
        }
 
        stage('Image Security Scan') {
            steps {
                sh '''
                  export DOCKER_IMAGE=${DOCKERHUB_USERNAME}/${APP_NAME}
                  trivy image --exit-code 0 \
                    --severity HIGH,CRITICAL \
                    --format table \
                    --output trivy-image-report.txt \
                    ${DOCKER_IMAGE}:${IMAGE_TAG}
                  cat trivy-image-report.txt
                '''
            }
            post {
                always { archiveArtifacts artifacts: 'trivy-image-report.txt', fingerprint: true }
            }
        }
 
        stage('Container Test') {
            steps {
                sh '''
                  export DOCKER_IMAGE=${DOCKERHUB_USERNAME}/${APP_NAME}
                  docker run -d --name test-${BUILD_NUMBER} -p 8888:80 ${DOCKER_IMAGE}:${IMAGE_TAG}
                  sleep 5
                  STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8888)
                  echo "HTTP status: $STATUS"
                  if [ "$STATUS" != "200" ]; then
                    docker logs test-${BUILD_NUMBER}
                    docker stop test-${BUILD_NUMBER} || true
                    docker rm test-${BUILD_NUMBER} || true
                    exit 1
                  fi
                  docker stop test-${BUILD_NUMBER}
                  docker rm test-${BUILD_NUMBER}
                  echo "Container test PASSED"
                '''
            }
        }
 
        stage('Push to DockerHub') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'DOCKERHUB_CREDS',
                                     usernameVariable: 'DH_USER',
                                     passwordVariable: 'DH_PASS')
                ]) {
                    sh '''
                      export DOCKER_IMAGE=${DOCKERHUB_USERNAME}/${APP_NAME}
                      echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                      docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                      docker push ${DOCKER_IMAGE}:latest
                      echo "Pushed: ${DOCKER_IMAGE}:${IMAGE_TAG}"
                    '''
                }
            }
        }
 
        // ── Deploy to STAGING (development branch only) ──
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'TARGET_VM_SSH',
                                      keyFileVariable: 'SSH_KEY',
                                      usernameVariable: 'VM_USER'),
                    string(credentialsId: 'TARGET_VM_HOST_STAGING', variable: 'VM_HOST')
                ]) {
                    sh '''
                      export DOCKER_IMAGE=${DOCKERHUB_USERNAME}/${APP_NAME}
                      echo "=== Deploying to STAGING ==="
                      ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no ${VM_USER}@${VM_HOST} << EOF
                        set -e
                        docker pull ${DOCKER_IMAGE}:${IMAGE_TAG}
                        docker stop ${APP_NAME} 2>/dev/null || true
                        docker rm ${APP_NAME} 2>/dev/null || true
                        docker run -d \
                          --name ${APP_NAME} \
                          --restart unless-stopped \
                          -p ${HOST_PORT}:${CONTAINER_PORT} \
                          ${DOCKER_IMAGE}:${IMAGE_TAG}
                        sleep 3
                        docker ps | grep ${APP_NAME}
                        echo "Staging deployment complete!"
EOF
                    '''
                }
            }
        }
 
        // ── Deploy to PRODUCTION (master branch only) ──
        stage('Deploy to Production') {
            when {
                branch 'dev'
            }
            steps {
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'TARGET_VM_SSH',
                                      keyFileVariable: 'SSH_KEY',
                                      usernameVariable: 'VM_USER'),
                    string(credentialsId: 'TARGET_VM_HOST_PROD', variable: 'VM_HOST')
                ]) {
                    sh '''
                      export DOCKER_IMAGE=${DOCKERHUB_USERNAME}/${APP_NAME}
                      echo "=== Deploying to PRODUCTION ==="
                      ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no ${VM_USER}@${VM_HOST} << EOF
                        set -e
                        docker pull ${DOCKER_IMAGE}:${IMAGE_TAG}
                        docker stop ${APP_NAME} 2>/dev/null || true
                        docker rm ${APP_NAME} 2>/dev/null || true
                        docker run -d \
                          --name ${APP_NAME} \
                          --restart unless-stopped \
                          -p ${HOST_PORT}:${CONTAINER_PORT} \
                          ${DOCKER_IMAGE}:${IMAGE_TAG}
                        sleep 3
                        docker ps | grep ${APP_NAME}
                        echo "Production deployment complete!"
EOF
                    '''
                }
            }
        }
    }
 
    post {
        success {
            echo "SUCCESS — Branch: ${env.BRANCH_NAME} | Build: #${BUILD_NUMBER} deployed"
        }
        failure {
            echo "FAILED — Branch: ${env.BRANCH_NAME} | Build: #${BUILD_NUMBER}"
        }
        always {
            sh 'docker image prune -f || true'
        }
    }
}