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
                git url: 'https://github.com/aparnaakhilesh/python-docker-project.git'
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'
            }
        }

        stage('Push Image') {
            steps {
                sh 'docker push ${IMAGE_NAME}:${IMAGE_TAG}'
            }
        }

        stage('Cleanup Old Container') {
            steps {
                sh '''
                  docker rm -f ${CONTAINER_NAME} || true
                '''
            }
        }

        stage('Run Container') {
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
}
``
