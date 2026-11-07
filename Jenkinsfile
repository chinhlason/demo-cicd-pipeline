pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    stages {
        stage('Validate onboarding config') {
            steps {
                script {
                    def required = [
                        'APP_REPO_URL',
                        'APP_BRANCH',
                        'DOCKER_IMAGE',
                        'GITOPS_REPO_URL',
                        'GITOPS_BRANCH',
                        'GITOPS_VALUES_FILE'
                    ]
                    def missing = required.findAll { !params[it]?.trim() }
                    if (missing) {
                        error "Missing Jenkins job parameters: ${missing.join(', ')}"
                    }
                    echo "Application repository: ${params.APP_REPO_URL} (${params.APP_BRANCH})"
                    echo "Docker image: ${params.DOCKER_IMAGE}"
                    echo "GitOps repository: ${params.GITOPS_REPO_URL} (${params.GITOPS_BRANCH})"
                    echo "GitOps values file: ${params.GITOPS_VALUES_FILE}"
                }
            }
        }

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

        stage('Update GitOps') {
            steps {
                script {
                    env.GITOPS_VALUES_FILE = params.GITOPS_VALUES_FILE
                    env.GITOPS_BRANCH = params.GITOPS_BRANCH
                    env.IMAGE_DIGEST = sh(
                        script: "docker image inspect --format='{{index .RepoDigests 0}}' '${env.IMAGE_TAG}' | sed 's#^.*@##'",
                        returnStdout: true
                    ).trim()
                }

                dir('gitops') {
                    git branch: params.GITOPS_BRANCH,
                        credentialsId: 'github-https',
                        url: params.GITOPS_REPO_URL

                    sh '''
                        set -eu
                        test -f "$GITOPS_VALUES_FILE"
                        DOCKER_IMAGE="$DOCKER_IMAGE" IMAGE_DIGEST="$IMAGE_DIGEST" \
                          python3 - "$GITOPS_VALUES_FILE" <<'PY'
import os
import pathlib
import re
import sys

path = pathlib.Path(sys.argv[1])
text = path.read_text()
text, repository_count = re.subn(
    r"(?m)^  repository: .+$",
    f"  repository: {os.environ['DOCKER_IMAGE']}",
    text,
    count=1,
)
text, digest_count = re.subn(
    r"(?m)^  digest: .+$",
    f"  digest: {os.environ['IMAGE_DIGEST']}",
    text,
    count=1,
)
if repository_count != 1 or digest_count != 1:
    raise SystemExit("expected one image repository and digest in values file")
path.write_text(text)
PY

                        git config user.name "jenkins"
                        git config user.email "jenkins@localhost"
                        git add "$GITOPS_VALUES_FILE"

                        if git diff --cached --quiet; then
                            echo "GitOps values already use $IMAGE_DIGEST"
                            exit 0
                        fi

                        git commit -m "deploy: update ${DOCKER_IMAGE} to ${IMAGE_DIGEST}"
                    '''

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'github-https',
                            usernameVariable: 'GITHUB_USER',
                            passwordVariable: 'GITHUB_TOKEN'
                        )
                    ]) {
                        sh '''
                            set -eu
                            ASKPASS="$PWD/.git-askpass"
                            cat > "$ASKPASS" <<'EOF'
#!/bin/sh
case "$1" in
  *Username*) printf '%s\n' "$GITHUB_USER" ;;
  *Password*) printf '%s\n' "$GITHUB_TOKEN" ;;
esac
EOF
                            chmod 700 "$ASKPASS"
                            GIT_ASKPASS="$ASKPASS" \
                              GIT_TERMINAL_PROMPT=0 \
                              git push origin "HEAD:$GITOPS_BRANCH"
                            rm -f "$ASKPASS"
                        '''
                    }
                }
            }
        }
    }
}
