pipeline {
    agent any

    environment {
        DOCKERHUB_USER = "aparnaakhilesh"
        IMAGE_NAME     = "python-cicd-app"
        CONTAINER_NAME = "python-app"
        DOCKER_CREDS   = credentials('dockerhub-creds')
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
                """
            }
        }

        /* -------------------------------------------------------
           DELETE OLD TAGS FROM DOCKER HUB (KEEP ONLY LAST 3)
        ---------------------------------------------------------*/
        stage("Cleanup Docker Hub old tags") {
            steps {
                script {
                    def KEEP1 = BUILD_NUMBER.toInteger()
                    def KEEP2 = KEEP1 - 1
                    def KEEP3 = KEEP1 - 2

                    echo "Keeping remote tags: ${KEEP1}, ${KEEP2}, ${KEEP3}"

                    // Read all tags via Docker Hub API
                    def json = sh(
                        script: """
                        curl -s -u ${DOCKERHUB_USER}:${DOCKER_CREDS_PSW} \
                        https://hub.docker.com/v2/repositories/${DOCKERHUB_USER}/${IMAGE_NAME}/tags/?page_size=100
                        """,
                        returnStdout: true
                    )

                    def tags = new groovy.json.JsonSlurper().parseText(json).results.collect { it.name }

                    for (tag in tags) {
                        if (tag.isInteger()) {
                            if (tag != KEEP1.toString() &&
                                tag != KEEP2.toString() &&
                                tag != KEEP3.toString()) {

                                echo "Deleting remote tag: ${tag}"

                                sh """
                                curl -s -X DELETE -u ${DOCKERHUB_USER}:${DOCKER_CREDS_PSW} \
                                https://hub.docker.com/v2/repositories/${DOCKERHUB_USER}/${IMAGE_NAME}/tags/${tag}/
                                """
                            }
                        }
                    }
                }
            }
        }

        /* -------------------------------------------------------
           DELETE OLD LOCAL DOCKER IMAGES (KEEP ONLY LAST 3)
        ---------------------------------------------------------*/
        stage("Cleanup local images and containers") {
            steps {
                script {

                    def KEEP1 = BUILD_NUMBER.toInteger()
                    def KEEP2 = KEEP1 - 1
                    def KEEP3 = KEEP1 - 2

                    echo "Keeping local builds: ${KEEP1}, ${KEEP2}, ${KEEP3}"

                    def allTags = sh(
                        script: "docker images ${DOCKERHUB_USER}/${IMAGE_NAME} --format '{{.Tag}}'",
                        returnStdout: true
                    ).trim().split("\n")

                    for (tag in allTags) {

                        if (tag != KEEP1.toString() &&
                            tag != KEEP2.toString() &&
                            tag != KEEP3.toString()) {

                            echo "Deleting local tag: ${tag}"

                            sh """
                            docker ps -a --filter ancestor=${DOCKERHUB_USER}/${IMAGE_NAME}:${tag} -q | xargs -r docker stop
                            docker ps -a --filter ancestor=${DOCKERHUB_USER}/${IMAGE_NAME}:${tag} -q | xargs -r docker rm
                            docker rmi -f ${DOCKERHUB_USER}/${IMAGE_NAME}:${tag} || true
                            """
                        }
                    }
                }
            }
        }

        /* -------------------------------------------------------
           RUN CONTAINER
        ---------------------------------------------------------*/
        stage("Run Container") {
            steps {
                sh """
                docker stop $CONTAINER_NAME || true
                docker rm $CONTAINER_NAME || true

                docker run -d \
                --name $CONTAINER_NAME \
                -p 5000:5000 \
                $DOCKERHUB_USER/$IMAGE_NAME:$BUILD_NUMBER
                """
            }
        }
    }

    post {
        success {
            echo "CI/CD Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed"
        }
    }
}
