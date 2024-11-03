pipeline {
    agent {
        label 'ec2-fleet'
    }

    options {
        // Keeps builds for the last 30 days.
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        // Prevents concurrent builds of the same pipeline.
        disableConcurrentBuilds()
        // Adds timestamps to the log output.
        timestamps()
    }

    environment {
        // Define environment variables
        AWS_REGION = 'us-east-2'
        ACCOUNT_ID = '023196572641'
        APP_IMAGE = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/app-repo:app:${BUILD_NUMBER}"
        CHART_VERSION = "0.1.${BUILD_NUMBER}"
        KUBECONFIG = "${env.WORKSPACE}/.kube/config"
    }

    stages {
        stage('ECR Login') {
            steps {
                sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
            }
        }
        stage('Pull and Tag Docker Image') {
            steps {
                sh """
                    docker build -t ${APP_IMAGE} .
                    docker tag ${APP_IMAGE} ${APP_IMAGE}:${BUILD_NUMBER}
                    docker push ${APP_IMAGE}:${BUILD_NUMBER}
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
                        --namespace ofri-test
                    """
                }
            }
        }
    }
}