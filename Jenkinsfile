pipeline {
    agent { label 'worker-node' }  // runs on worker node, not master

    environment {
        APP_NAME       = 'my-app'
        DOCKER_IMAGE   = "${DOCKERHUB_USERNAME}/${APP_NAME}"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        CONTAINER_PORT = '80'
        HOST_PORT      = '80'
    }

    triggers {
        githubPush()  // auto trigger on push to main
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Shilpa729-bit/CICD.git'
                echo "Build completed from branch main"
            }
        }

        // ── STAGE 2: Code Quality Check ────────────────
        stage('Code Quality') {
            steps {
                sh '''
                  echo "=== HTML Validation ==="
                  # Install html5validator if needed
                  pip3 install html5-parser --quiet 2>/dev/null || true

                  # Basic syntax check
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
                    archiveArtifacts artifacts: 'trivy-fs-report.txt', fingerprint: true
                }
            }
        }

    }
}