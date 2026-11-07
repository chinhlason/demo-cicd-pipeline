pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    parameters {
        string(
            name: 'APP_REPO_URL',
            defaultValue: 'https://github.com/chinhlason/demo-cicd-backend-app.git',
            description: 'HTTPS URL of the application repository'
        )
        string(
            name: 'APP_BRANCH',
            defaultValue: 'main',
            description: 'Application branch to build'
        )
        string(
            name: 'DOCKER_IMAGE',
            defaultValue: 'chinhlason/demo-cicd-backend-app',
            description: 'Docker Hub image name without the tag'
        )
    }

    stages {
        stage('Checkout application') {
            steps {
                dir('app') {
                    git branch: params.APP_BRANCH,
                        credentialsId: 'github-https',
                        url: params.APP_REPO_URL
                }
            }
        }

        stage('Test') {
            steps {
                dir('app') {
                    sh './mvnw test -q'
                }
            }
        }

        stage('Package') {
            steps {
                dir('app') {
                    sh './mvnw package -q -DskipTests'
                }
            }
        }

        stage('Build and push image') {
            steps {
                dir('app') {
                    script {
                        def appCommit = sh(
                            script: 'git rev-parse --short=12 HEAD',
                            returnStdout: true
                        ).trim()
                        env.IMAGE_TAG = "${params.DOCKER_IMAGE}:${appCommit}"
                    }

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_TOKEN'
                        )
                    ]) {
                        sh '''
                            set -eu
                            printf '%s' "$DOCKER_TOKEN" | docker login \
                                --username "$DOCKER_USER" \
                                --password-stdin
                            docker build --tag "$IMAGE_TAG" .
                            docker push "$IMAGE_TAG"
                            docker logout
                        '''
                    }
                }
            }
        }
    }
}
