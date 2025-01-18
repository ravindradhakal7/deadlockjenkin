pipeline {
    agent any

    environment {
        // DockerHub credentials stored in Jenkins Credentials
        DOCKER_CREDENTIALS = 'docker-hub-credentials'
        DOCKER_IMAGE = 'ravindradhakal/deadlocktest'
        VERSION = '1.0.1' // This can be parameterized if needed
    }

    parameters {
        string(name: 'VERSION', defaultValue: '1.0.1', description: 'Version of the Docker image')
    }

    stages {
        stage('Checkout Code') {
            steps {
                    echo 'Fetching code from GitHub repository.'
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: '*/main']], // Replace 'main' with your branch name
                        doGenerateSubmoduleConfigurations: false,
                        extensions: [],
                        userRemoteConfigs: [[
                            url: 'https://github.com/ravindradhakal7/deadlocktest.git',
                            credentialsId: '32a54451-4069-4121-adf4-0ff59722bf19' // Use the ID of the credential you added
                        ]]
                    ])
                    echo 'Code checkout complete.'
                }
        }

        stage('Build') {
            steps {
                // Build the Docker image
                script {
                    docker.build("${DOCKER_IMAGE}:${VERSION}")
                }
            }
        }

        stage('Login to DockerHub') {
            steps {
                // Login to DockerHub using Jenkins credentials
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CREDENTIALS) {
                        echo 'Logged into DockerHub'
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                // Push the Docker image to DockerHub
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CREDENTIALS) {
                        docker.image("${DOCKER_IMAGE}:${VERSION}").push()
                    }
                }
            }
        }

        stage('Cleanup') {
            steps {
                // Remove the local Docker image to free up space
                script {
                    sh "docker rmi ${DOCKER_IMAGE}:${VERSION}"
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up after build'
        }

        success {
            echo 'Build and Docker image push successful!'
        }

        failure {
            echo 'Build or push failed.'
        }
    }
}
