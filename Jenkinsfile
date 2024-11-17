pipeline {
    agent {
        label 'jenkins/release-jenkins-jenkins-agent'
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        AWS_REGION = 'us-east-2'  // Your AWS region
        ACCOUNT_ID = '023196572641' // Your AWS account ID
        ECR_REPO = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/ronn4/app-repo"
        IMAGE_TAG = "${BUILD_NUMBER}"
        APP_IMAGE = "${ECR_REPO}:app-${IMAGE_TAG}"
        CHART_VERSION = "0.2.${BUILD_NUMBER}"
        KUBECONFIG = "${env.WORKSPACE}/.kube/config"
        DOCKER_REPO = 'ronn4/repo1'
        DOCKERHUB_CREDENTIALS = 'dockerhub'
        NAMESPACE = 'ronn4-jenkins'
        APP_IMAGE_NAME = 'app-image-latest'
        WEB_IMAGE_NAME = 'web-image-latest'
    }

    stages {
        stage('Checkout and Extract Git Commit Hash') {
            steps {
                checkout scm
                script {
                    sh 'git rev-parse --short HEAD > gitCommit.txt'
                    def GITCOMMIT = readFile('gitCommit.txt').trim()
                    env.GIT_TAG = "${GITCOMMIT}"
                    env.IMAGE_TAG = "v1.0.0-${BUILD_NUMBER}-${GIT_TAG}"
                }
            }
        }

        stage('Install Python Requirements') {
            steps {
                sh """
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install pytest unittest2 pylint flask telebot Pillow loguru matplotlib
                """
            }
        }

        stage('Unittest') {
            steps {
                sh """
                    python3 -m venv venv
                    . venv/bin/activate
                    python -m pytest --junitxml results.xml polybot/test
                """
            }
        }

        stage('ECR Login') {
            steps {
                script {
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}"
                }
            }
        }

        stage('Login, Tag, and Push Images to ECR') {
            steps {
                script {
                    sh """
                    cd polybot
                    docker build -t ${APP_IMAGE_NAME}:latest .
                    docker tag ${APP_IMAGE_NAME}:latest ${APP_IMAGE}
                    docker push ${APP_IMAGE}
                    """
                }
            }
        }

        stage('Package Helm Chart') {
            steps {
                script {
                    sh """
                        sed -i 's/^version:.*/version: ${CHART_VERSION}/' my-python-app-chart/Chart.yaml
                        helm package ./my-python-app-chart
                    """
                }
            }
        }

        stage('Deploy with Helm') {
            steps {
                script {
                    withEnv(["KUBECONFIG=/home/ec2-user/.kube/config"]) {
                        sh """
                            helm upgrade --install my-python-app ./my-python-app-${CHART_VERSION}.tgz \
                            --set image.tag=app-${BUILD_NUMBER} \
                            --namespace ronn4-test
                        """
                    }
                }
            }
        }

        stage('Publish to SNS') {
            steps {
                script {
                    sh 'aws sns publish --topic-arn arn:aws:sns:us-east-2:023196572641:ronn4-sns --message "Build Notification from Jenkins"'
                }
            }
        }
    }

    post {
        success {
            script {
                sh """
                    aws sns publish --topic-arn arn:aws:sns:us-east-2:023196572641:ronn4-sns --message "Build succeeded for ${env.JOB_NAME} #${env.BUILD_NUMBER}" --subject "Jenkins Build Success"
                """
            }
        }
        failure {
            script {
                sh """
                    aws sns publish --topic-arn arn:aws:sns:us-east-2:023196572641:ronn4-sns --message "Build failed for ${env.JOB_NAME} #${env.BUILD_NUMBER}" --subject "Jenkins Build Failure"
                """
            }
        }
        always {
            cleanWs() // Clean workspace after every build, regardless of success or failure
        }
    }
}
