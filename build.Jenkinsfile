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
        APP_IMAGE_NAME = 'app-image'
        WEB_IMAGE_NAME = 'web-image'
        DOCKER_COMPOSE_FILE = 'compose.yaml'
        AWS_REGION = 'us-east-2'
        ACCOUNT_ID = '023196572641'
        APP_IMAGE = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/app-repo:app-image-0.3.${BUILD_NUMBER}"
        CHART_VERSION = "0.1.${BUILD_NUMBER}"
        KUBECONFIG = "${env.WORKSPACE}/.kube/config"
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
                sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
            }
        }
        stage('Login, Tag, and Push Images') {
            steps {
                script {
                    // Login to Dockerhub, tag, and push images
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
                    withEnv(["KUBECONFIG=${env.KUBECONFIG}"]) {
                        sh """
                            helm upgrade --install my-python-app ./my-python-app-${CHART_VERSION}.tgz \
                            --set image.tag=${BUILD_NUMBER} \
                            --atomic --wait \
                            --namespace ofri-test
                        """
                    }
                }
            }
        }
    }
}