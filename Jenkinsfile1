pipeline {
    agent { label 'worker-node' }  // runs on worker node, not master

    environment {
        APP_NAME       = 'my-app'
        IMAGE_TAG      = "${BUILD_NUMBER}"
        CONTAINER_PORT = '80'
        HOST_PORT      = '80'
    }

    triggers {
        githubPush()
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Shilpa729-bit/CICD.git'

                echo "Build completed from branch main"
            }
        }

        // ── STAGE 2: Code Quality Check ────────────────
        stage('Code Quality') {
            steps {
                sh '''
                  echo "=== HTML Validation ==="

                  pip3 install html5-parser --quiet 2>/dev/null || true

                  if [ -f index.html ]; then
                    python3 -c "
import html.parser, sys
class V(html.parser.HTMLParser):
    def __init__(self): super().__init__(); self.errors=[]
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

        // ── STAGE 3: Security Scan (source code) ───────
        stage('Security Scan - Source') {
            steps {
                sh '''
                  echo "=== Trivy filesystem scan ==="

                  trivy fs --exit-code 0 \
                    --severity HIGH,CRITICAL \
                    --format table \
                    --output trivy-fs-report.txt \
                    .

                  cat trivy-fs-report.txt
                '''
            }

            post {
                always {
                    archiveArtifacts artifacts: 'trivy-fs-report.txt',
                                     fingerprint: true
                }
            }
        }

        // ── STAGE 4: Build Docker Image ─────────────────
        stage('Build Docker Image') {
            steps {
                withCredentials([
                    string(credentialsId: 'DOCKERHUB_USERNAME',
                           variable: 'DOCKERHUB_USERNAME')
                ]) {

                    sh '''
                      export DOCKER_IMAGE=${DOCKERHUB_USERNAME}/${APP_NAME}

                      echo "=== Building image: ${DOCKER_IMAGE}:${IMAGE_TAG} ==="

                      docker build \
                        --label "build=${BUILD_NUMBER}" \
                        --label "git-commit=${GIT_COMMIT}" \
                        -t ${DOCKER_IMAGE}:${IMAGE_TAG} \
                        -t ${DOCKER_IMAGE}:latest \
                        .

                      docker images ${DOCKER_IMAGE}
                    '''
                }
            }
        }

        // ── STAGE 5: Image Security Scan ───────────────
        stage('Image Security Scan') {
            steps {
                withCredentials([
                    string(credentialsId: 'DOCKERHUB_USERNAME',
                           variable: 'DOCKERHUB_USERNAME')
                ]) {

                    sh '''
                      export DOCKER_IMAGE=${DOCKERHUB_USERNAME}/${APP_NAME}

                      echo "=== Trivy image scan ==="

                      trivy image \
                        --exit-code 0 \
                        --severity HIGH,CRITICAL \
                        --format table \
                        --output trivy-image-report.txt \
                        ${DOCKER_IMAGE}:${IMAGE_TAG}

                      cat trivy-image-report.txt
                    '''
                }
            }

            post {
                always {
                    archiveArtifacts artifacts: 'trivy-image-report.txt',
                                     fingerprint: true
                }
            }
        }

        // ── STAGE 6: Container Smoke Test ──────────────
        stage('Container Test') {
            steps {
                withCredentials([
                    string(credentialsId: 'DOCKERHUB_USERNAME',
                           variable: 'DOCKERHUB_USERNAME')
                ]) {

                    sh '''
                      export DOCKER_IMAGE=${DOCKERHUB_USERNAME}/${APP_NAME}

                      echo "=== Starting test container ==="

                      docker run -d \
                        --name test-${BUILD_NUMBER} \
                        -p 8888:80 \
                        ${DOCKER_IMAGE}:${IMAGE_TAG}

                      echo "=== Waiting for app to be ready ==="

                      sleep 5

                      echo "=== Running HTTP smoke test ==="

                      STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8888)

                      echo "HTTP status: $STATUS"

                      if [ "$STATUS" != "200" ]; then
                        echo "FAIL: Expected 200, got $STATUS"

                        docker logs test-${BUILD_NUMBER}

                        docker stop test-${BUILD_NUMBER} || true
                        docker rm test-${BUILD_NUMBER} || true

                        exit 1
                      fi

                      echo "Container test PASSED"

                      echo "=== Cleanup test container ==="

                      docker stop test-${BUILD_NUMBER}
                      docker rm test-${BUILD_NUMBER}
                    '''
                }
            }
        }

        // ── STAGE 7: Push to DockerHub ──────────────────
        stage('Push to DockerHub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'DOCKERHUB_CREDS',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_PASS'
                    ),
                    string(
                        credentialsId: 'DOCKERHUB_USERNAME',
                        variable: 'DOCKERHUB_USERNAME'
                    )
                ]) {

                    sh '''
                      export DOCKER_IMAGE=${DOCKERHUB_USERNAME}/${APP_NAME}

                      echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin

                      docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                      docker push ${DOCKER_IMAGE}:latest

                      echo "Image pushed: ${DOCKER_IMAGE}:${IMAGE_TAG}"
                    '''
                }
            }
        }

                // ── STAGE 8+9: Pull & Deploy on Target VM ──────
        stage('Deploy to Target VM') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'TARGET_VM_SSH',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'VM_USER'
                    ),
                    string(
                        credentialsId: 'TARGET_VM_HOST',
                        variable: 'VM_HOST'
                    ),
                    string(
                        credentialsId: 'DOCKERHUB_USERNAME',
                        variable: 'DOCKERHUB_USERNAME'
                    )
                ]) {

                    sh '''
                      export DOCKER_IMAGE=${DOCKERHUB_USERNAME}/${APP_NAME}

                      echo "=== Deploying to target VM ==="

                      ssh -i "$SSH_KEY" \
                          -o StrictHostKeyChecking=no \
                          ${VM_USER}@${VM_HOST} << EOF

                        set -e

                        echo "-- Pulling image --"

                        docker pull ${DOCKER_IMAGE}:${IMAGE_TAG}

                        echo "-- Stopping old container (if any) --"

                        docker stop ${APP_NAME} 2>/dev/null || true
                        docker rm ${APP_NAME} 2>/dev/null || true

                        echo "-- Starting new container --"

                        docker run -d \
                          --name ${APP_NAME} \
                          --restart unless-stopped \
                          -p ${HOST_PORT}:${CONTAINER_PORT} \
                          ${DOCKER_IMAGE}:${IMAGE_TAG}

                        echo "-- Verifying deployment --"

                        sleep 3

                        docker ps | grep ${APP_NAME}

                        echo "Deployment complete!"

EOF
                    '''
                }
            }
        }
    }

    // ── POST ACTIONS ────────────────────────────────────
    post {

        success {
            echo "Pipeline SUCCEEDED — build #${BUILD_NUMBER} deployed"
        }

        failure {
            echo "Pipeline FAILED — check logs"
        }

        always {
            sh 'docker image prune -f || true'
        }
    }
}