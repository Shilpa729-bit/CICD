pipeline {
    agent { label 'worker-node' }  // runs on worker node, not master

    environment {
        APP_NAME        = 'my-app'
        DOCKER_IMAGE    = "${DOCKERHUB_USERNAME}/${APP_NAME}"
        IMAGE_TAG       = "${BUILD_NUMBER}"
        CONTAINER_PORT  = '80'
        HOST_PORT       = '80'
    }

    triggers {
        githubPush()  // auto trigger on push to main
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Shilpa729-bit/CICD.git'
                echo "Building commit: ${GIT_COMMIT[0..7]}"
            }
        }

    }
}

    