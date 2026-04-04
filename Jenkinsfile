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
                docker tag $DOCKERHUB_USER/$IMAGE_NAME:$BUILD_NUMBER $DOCKERHUB_USER/$IMAGE_NAME:latest
                docker push $DOCKERHUB_USER/$IMAGE_NAME:latest
                """
            }
        }

        /* ---------------------------------------------------
           CLEANUP ON DOCKER HUB (remote registry)
           Keep: latest + last 3 builds
        ----------------------------------------------------*/
        stage("Cleanup Docker Hub old tags") {
            steps {
                script {

                    def KEEP1 = BUILD_NUMBER.toInteger()
                    def KEEP2 = KEEP1 - 1
                    def KEEP3 = KEEP1 - 2

                    echo "Keeping remote tags: latest, ${KEEP1}, ${KEEP2}, ${KEEP3}"

                    def json = sh(
                        script: """
                        curl -s -u ${DOCKERHUB_USER}:${DOCKER_CREDS_PSW} \
                        https://hub.docker.com/v2/repositories/${DOCKERHUB_USER}/${IMAGE_NAME}/tags/?page_size=100
                        """,
                        returnStdout: true
                    )

                    def tags = new groovy.json.JsonSlurper().parseText(json).results.collect { it.name }

                    for (tag in tags) {

                        if (tag == "latest") continue

                        if (tag != KEEP1.toString() &&
                            tag != KEEP2.toString() &&
                            tag != KEEP3.toString()) {

                            echo "Deleting remote tag from Docker Hub: ${tag}"

                            sh """
                            curl -s -X DELETE -u ${DOCKERHUB_USER}:${DOCKER_CREDS_PSW} \
                            https://hub.docker.com/v2/repositories/${DOCKERHUB_USER}/${IMAGE_NAME}/tags/${tag}/
                            """
                        }
                    }
                }
            }
        }
        
        stage('Cleanup Docker Hub old tags') {
    steps {
        script {

            // Keep these tags
            def keepTags = ["latest", "${env.IMAGE_TAG}", "${env.IMAGE_TAG - 1}", "${env.IMAGE_TAG - 2}"]
            echo "Keeping remote tags: ${keepTags.join(', ')}"

            withCredentials([usernamePassword(
                credentialsId: 'dockerhub-creds', 
                usernameVariable: 'DH_USER', 
                passwordVariable: 'DH_TOKEN'
            )]) {

                // Get all tags from Docker Hub
                def response = sh(
                    script: """
                        curl -s -u "$DH_USER:$DH_TOKEN" \
                        "https://hub.docker.com/v2/repositories/aparnaakhilesh/python-cicd-app/tags/?page_size=100"
                    """,
                    returnStdout: true
                ).trim()

                def json = readJSON text: response
                def allTags = json.results*.name

                // Determine which tags to delete
                def deleteTags = allTags.findAll { !(it in keepTags) }

                deleteTags.each { tag ->
                    echo "Deleting remote tag from Docker Hub: ${tag}"

                    // Fetch the digest for that tag
                    def tagInfo = sh(
                        script: """
                            curl -s -u "$DH_USER:$DH_TOKEN" \
                            "https://hub.docker.com/v2/repositories/aparnaakhilesh/python-cicd-app/tags/${tag}/"
                        """,
                        returnStdout: true
                    ).trim()

                    def tagJson = readJSON text: tagInfo
                    def digest = tagJson.images[0]?.digest

                    if (!digest) {
                        echo "Could not retrieve digest for tag: ${tag}, skipping"
                        return
                    }

                    // Delete via digest
                    sh """
                        curl -s -X DELETE -u "$DH_USER:$DH_TOKEN" \
                        "https://hub.docker.com/v2/repositories/aparnaakhilesh/python-cicd-app/tags/${digest}/"
                    """

                    echo "Deleted: ${tag}"
                }
            }
        }
    }
}

        /* ---------------------------------------------------
           LOCAL CLEANUP (docker engine)
           Keep: latest + last 3 builds
        ----------------------------------------------------*/
        stage("Cleanup local images and containers") {
            steps {
                script {

                    def KEEP1 = BUILD_NUMBER.toInteger()
                    def KEEP2 = KEEP1 - 1
                    def KEEP3 = KEEP1 - 2

                    echo "Keeping local tags: latest, ${KEEP1}, ${KEEP2}, ${KEEP3}"

                    def allTags = sh(
                        script: "docker images ${DOCKERHUB_USER}/${IMAGE_NAME} --format '{{.Tag}}'",
                        returnStdout: true
                    ).trim().split("\n")

                    for (tag in allTags) {

                        if (tag == "latest") continue

                        if (tag != KEEP1.toString() &&
                            tag != KEEP2.toString() &&
                            tag != KEEP3.toString()) {

                            echo "Removing local tag: ${tag}"

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

        /* ---------------------------------------------------
           RUN FINAL CONTAINER
        ----------------------------------------------------*/
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
        success { echo "CI/CD Pipeline completed successfully!" }
        failure { echo "Pipeline failed" }
    }
}
