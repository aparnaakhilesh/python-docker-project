pipeline {
    agent any

    environment {
        IMAGE_NAME     = "aparnaakhilesh/python-docker-project"
        IMAGE_TAG      = "latest"
        CONTAINER_NAME = "python-app"
        APP_PORT       = "5000"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/aparnaakhilesh/python-docker-project.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                  docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                  docker push ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Stop & Remove Old Container') {
            steps {
                sh '''
                  docker stop ${CONTAINER_NAME} || true
                  docker rm ${CONTAINER_NAME} || true
                '''
            }
        }

        stage('Run New Container') {
            steps {
                sh '''
                  docker run -d \
                    --name ${CONTAINER_NAME} \
                    -p ${APP_PORT}:${APP_PORT} \
                    ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Build, push, and run completed successfully"
        }
        failure {
            echo "❌ Pipeline failed"
        }
    }
}
