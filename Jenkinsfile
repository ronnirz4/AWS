pipeline {
    agent {
        label 'ec2-fleet'
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        AWS_REGION = 'us-east-2'  // Your AWS region
        ACCOUNT_ID = '023196572641' // Your AWS account ID
        // Set ECR_REPO to the full URI of your ECR repository
        ECR_REPO = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/ronn4/app-repo"
        IMAGE_TAG = "${BUILD_NUMBER}"
        APP_IMAGE = "${ECR_REPO}:app-${IMAGE_TAG}"
        CHART_VERSION = "0.28.${BUILD_NUMBER}"
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
                // Checkout code
                checkout scm

                // Extract Git commit hash
                script {
                    sh 'git rev-parse --short HEAD > gitCommit.txt'
                    def GITCOMMIT = readFile('gitCommit.txt').trim()
                    env.GIT_TAG = "${GITCOMMIT}"

                    // Set IMAGE_TAG as an environment variable
                    env.IMAGE_TAG = "v1.0.0-${BUILD_NUMBER}-${GIT_TAG}"
                }
            }
        }

        stage('Install Python Requirements') {
            steps {
                // Install Python dependencies
                sh """
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install pytest unittest2 pylint flask telebot Pillow loguru matplotlib
                """
            }
        }

        stage('Unittest') {
            steps {
                // Run unittests
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
                    // Build the Docker image and push to ECR
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
                    // Update the chart version in Chart.yaml and package the chart
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
                            --set image.tag=app-image-28.0.${BUILD_NUMBER} \
                            --namespace ronn4-test
                        """
                    }
                }
            }
        }
    }
}
