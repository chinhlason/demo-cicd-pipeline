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
    }
}
