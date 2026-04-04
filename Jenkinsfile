pipeline {
    agent any

    environment {
        IMAGE_NAME = "aparnaakhilesh/python-cicd-app"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git url: 'https://github.com/aparnaakhilesh/python-docker-project.git', branch: 'main'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {

                    // Fetch last tag WITHOUT jq
                    def response = sh(
                        script: """
                            curl -s "https://hub.docker.com/v2/repositories/${IMAGE_NAME}/tags/?page_size=1"
                        """,
                        returnStdout: true
                    ).trim()

                    def json = readJSON text: response
                    def lastTag = json?.results ? json.results[0].name : "0"

                    // Increment tag
                    env.IMAGE_TAG = (lastTag.isInteger() ? lastTag.toInteger() + 1 : 1).toString()
                    echo "Building new image with tag: ${env.IMAGE_TAG}"

                    sh """
                        docker build -t ${IMAGE_NAME}:${env.IMAGE_TAG} .
                    """
                }
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_TOKEN'
                )]) {
                    sh """
                        echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                    """
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    sh "docker push ${IMAGE_NAME}:${env.IMAGE_TAG}"
                    sh "docker tag ${IMAGE_NAME}:${env.IMAGE_TAG} ${IMAGE_NAME}:latest"
                    sh "docker push ${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Cleanup Docker Hub Old Tags') {
            steps {
                script {

                    def keepTags = ["latest", env.IMAGE_TAG, (env.IMAGE_TAG.toInteger() - 1).toString(), (env.IMAGE_TAG.toInteger() - 2).toString()]
                    echo "Keeping remote tags: ${keepTags.join(', ')}"

                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )]) {

                        // Get tags
                        def resp = sh(
                            script: """
                                curl -s -u "$DH_USER:$DH_TOKEN" \
                                "https://hub.docker.com/v2/repositories/${IMAGE_NAME}/tags/?page_size=100"
                            """,
                            returnStdout: true
                        ).trim()

                        def respJson = readJSON text: resp
                        def allTags = respJson.results*.name

                        def deleteTags = allTags.findAll { !(it in keepTags) }

                        deleteTags.each { tag ->

                            echo "Deleting remote tag: ${tag}"

                            def tagInfo = sh(
                                script: """
                                    curl -s -u "$DH_USER:$DH_TOKEN" \
                                    "https://hub.docker.com/v2/repositories/${IMAGE_NAME}/tags/${tag}/"
                                """,
                                returnStdout: true
                            ).trim()

                            def tagJson = readJSON text: tagInfo
                            def digest = tagJson.images[0]?.digest

                            if (!digest) {
                                echo "Skipping - no digest found for: ${tag}"
                                return
                            }

                            sh """
                                curl -s -X DELETE -u "$DH_USER:$DH_TOKEN" \
                                "https://hub.docker.com/v2/repositories/${IMAGE_NAME}/tags/${digest}/"
                            """

                            echo "Deleted tag: ${tag}"
                        }
                    }
                }
            }
        }

        stage('Cleanup Local Images') {
            steps {
                script {
                    echo "Cleaning local Docker images..."

                    def keepTags = ["latest", env.IMAGE_TAG, (env.IMAGE_TAG.toInteger() - 1).toString(), (env.IMAGE_TAG.toInteger() - 2).toString()]
                    def localTags = sh(
                        script: "docker images ${IMAGE_NAME} --format '{{.Tag}}'",
                        returnStdout: true
                    ).trim().split("\n")

                    localTags.each { tag ->
                        if (!(tag in keepTags)) {
                            echo "Removing local tag: ${tag}"
                            sh """
                                docker ps -a --filter ancestor=${IMAGE_NAME}:${tag} -q | xargs -r docker stop
                                docker ps -a --filter ancestor=${IMAGE_NAME}:${tag} -q | xargs -r docker rm
                                docker rmi -f ${IMAGE_NAME}:${tag}
                            """
                        }
                    }
                }
            }
        }

        stage('Run Container') {
            steps {
                script {
                    sh "docker stop python-app || true"
                    sh "docker rm python-app || true"
                    sh "docker run -d --name python-app -p 5000:5000 ${IMAGE_NAME}:${env.IMAGE_TAG}"
                }
            }
        }
    }

    post {
        success {
            echo "CI/CD Pipeline completed successfully!"
        }
    }
}
