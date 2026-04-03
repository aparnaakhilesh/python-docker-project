pipeline {
    agent any

    environment {
        IMAGE_NAME = "aparnaakhilesh/python-docker-project"
        IMAGE_TAG  = "latest"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git url: 'https://github.com/aparnaakhilesh/python-docker-project.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                  docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                sh '''
                  docker push ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }
    }
}
