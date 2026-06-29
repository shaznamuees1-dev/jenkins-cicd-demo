pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'shaznamuees/jenkins-cicd-demo'
        BUILD_TAG = "${env.BUILD_NUMBER}"
        EC2_HOST = '47.129.120.73'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Lint & Test') {
            parallel {
                stage('Lint') {
                    steps {
                        sh '''
                            python3 -m venv venv
                            . venv/bin/activate
                            pip install flake8
                            flake8 app/ --max-line-length=120 || true
                        '''
                    }
                }
                stage('Test') {
                    steps {
                        sh '''
                            python3 -m venv venv
                            . venv/bin/activate
                            pip install -r requirements.txt
                            pytest tests/ -v
                        '''
                    }
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${BUILD_TAG} ."
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                    sh "docker push ${DOCKER_IMAGE}:${BUILD_TAG}"
                }
            }
        }

        stage('Deploy') {
    when {
        branch 'main'
    }
    steps {
        sshagent(['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} '
                        docker pull ${DOCKER_IMAGE}:${BUILD_TAG} &&
                        docker stop jenkins-demo-app || true &&
                        docker rm jenkins-demo-app || true &&
                        docker run -d --name jenkins-demo-app -p 8000:8000 ${DOCKER_IMAGE}:${BUILD_TAG}
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_URL')]) {
                sh """
                    curl -X POST -H 'Content-type: application/json' --data '{"text":"Build #${BUILD_NUMBER} succeeded for jenkins-cicd-demo"}' \$SLACK_URL
                """
            }
        }
        failure {
            withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_URL')]) {
                sh """
                    curl -X POST -H 'Content-type: application/json' --data '{"text":"Build #${BUILD_NUMBER} failed for jenkins-cicd-demo"}' \$SLACK_URL
                """
            }
        }
    }
}