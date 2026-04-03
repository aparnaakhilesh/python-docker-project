pipeline {
    agent any

    environment {
        DOCKERHUB_USER = "aparnaakhilesh"
        IMAGE_NAME = "python-cicd-app"
        CONTAINER_NAME = "python-app"
        DOCKER_CREDS = credentials('dockerhub-creds')
    }

    stages {

        stage("Checkout Code") {
            steps {
                git branch: 'main',
                    url: 'https://github.com/aparnaakhilesh/python-docker-project.git'
            }
        }

        stage("Build Docker Image") {
            steps {
                sh """
                docker build -t $DOCKERHUB_USER/$IMAGE_NAME:$BUILD_NUMBER .
                """
            }
        }

        stage("Login to Docker Hub") {
            steps {
                sh """
                echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin
                """
            }
        }

        stage("Push Docker Image") {
            steps {
                sh """
                docker push $DOCKERHUB_USER/$IMAGE_NAME:$BUILD_NUMBER
                docker tag $DOCKERHUB_USER/$IMAGE_NAME:$BUILD_NUMBER $DOCKERHUB_USER/$IMAGE_NAME:latest
                docker push $DOCKERHUB_USER/$IMAGE_NAME:latest
                """
            }
        }

        stage("Run Container") {
            steps {
                sh """
                docker stop $CONTAINER_NAME || true
                docker rm $CONTAINER_NAME || true

                docker run -d \
                --name $CONTAINER_NAME \
                -p 5000:5000 \
                $DOCKERHUB_USER/$IMAGE_NAME:latest
                """
            }
        }
    }

    post {
        success {
            echo "✅ CI/CD Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed"
        }
    }
}
