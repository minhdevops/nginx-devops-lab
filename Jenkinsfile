pipeline {
    agent any

    parameters {
        string(
            name: 'IMAGE_TAG',
            defaultValue: '',
            description: 'Để trống để Jenkins tự tạo tag 0.0.BUILD_NUMBER'
        )
    }

    environment {
        IMAGE_REPOSITORY = 'dovanminh98/nginx-demo'
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-creds'
        GITHUB_CREDENTIALS_ID = 'github-creds'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Prepare tag') {
            steps {
                script {
                    env.EFFECTIVE_TAG = params.IMAGE_TAG?.trim()
                        ? params.IMAGE_TAG.trim()
                        : "0.0.${env.BUILD_NUMBER}"
                }
                echo "Building ${IMAGE_REPOSITORY}:${EFFECTIVE_TAG}"
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build -t ${IMAGE_REPOSITORY}:${EFFECTIVE_TAG} .'
            }
        }

        stage('Push image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: env.DOCKERHUB_CREDENTIALS_ID,
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                        set +x
                        echo "$DOCKERHUB_TOKEN" | docker login \
                          --username "$DOCKERHUB_USER" \
                          --password-stdin
                        docker push "${IMAGE_REPOSITORY}:${EFFECTIVE_TAG}"
                    '''
                }
            }
        }

        stage('Update Helm tag') {
            steps {
                sh '''
                    sed -i "/^image:/,/^[^ ]/ s/^  tag:.*/  tag: \"${EFFECTIVE_TAG}\"/" \
                      helm/nginx-demo/values.yaml

                    git config user.name "Jenkins CI"
                    git config user.email "jenkins@lab.local"
                    git add helm/nginx-demo/values.yaml

                    if ! git diff --cached --quiet; then
                      git commit -m "chore: deploy nginx-demo ${EFFECTIVE_TAG} [skip ci]"
                    fi
                '''
            }
        }

        stage('Push GitOps change') {
            steps {
                withCredentials([gitUsernamePassword(
                    credentialsId: env.GITHUB_CREDENTIALS_ID,
                    gitToolName: 'Default'
                )]) {
                    sh 'git push origin HEAD:main'
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
        success {
            echo "Hoàn tất: ${IMAGE_REPOSITORY}:${EFFECTIVE_TAG}"
        }
    }
}

