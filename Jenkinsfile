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
        AWS_REGION = 'us-east-2'
        ACCOUNT_ID = '023196572641'
        ECR_REPO = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/app-repo"
        APP_IMAGE = "${ECR_REPO}:app"
        IMAGE_TAG = "${BUILD_NUMBER}"
        CHART_VERSION = "0.1.${BUILD_NUMBER}"
        KUBECONFIG = "${env.WORKSPACE}/.kube/config"
        DOCKER_REPO = 'ronn4/repo1'
        DOCKERHUB_CREDENTIALS = 'dockerhub'
        NAMESPACE = 'ronn4-jenkins'
    }

    stages {
        stage('ECR Login') {
            steps {
                script {
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}"
                }
            }
        }
        stage('Build, Tag, and Push Docker Image') {
            steps {
                sh """
                    docker build -t ${APP_IMAGE}:${IMAGE_TAG} .
                    docker push ${APP_IMAGE}:${IMAGE_TAG}
                """
            }
        }
        stage('Package and Deploy Helm Chart') {
            steps {
                script {
                    sh """
                        sed -i 's/^version:.*/version: ${CHART_VERSION}/' my-python-app-chart/Chart.yaml
                        helm package ./my-python-app-chart
                        helm upgrade --install my-python-app ./my-python-app-${CHART_VERSION}.tgz \
                        --atomic --wait \
                        --namespace ${NAMESPACE}
                    """
                }
            }
        }
    }
}
