pipeline {
    agent any

    environment {
        VERSION = "${BUILD_NUMBER}"
        CHART_VERSION = "0.1.${BUILD_NUMBER}"
        HELM_CHART_NAME = "helm-api-ui"
        HELM_RELEASE_NAME = "api-ui"
        NAMESPACE = "default"
        ARTIFACTORY_HELM_REPO = "https://trialbkmv1q.jfrog.io/artifactory/api/helm/api-ui-helm"
        IMAGE_API = "guna5502/api-server:${VERSION}"
        IMAGE_UI  = "guna5502/react-ui:${VERSION}"

        ART_USER = credentials('artifactory-username')
        ART_PASS = credentials('artifactory-password')
        DOCKER_USER = credentials('dockerhub-username')
        DOCKER_PASS = credentials('dockerhub-password')
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
                sh 'pwd && ls -l'
            }
        }

        stage('Build Docker Images') {
            steps {
                dir('api-server-flask') {
                    sh "docker build -t ${IMAGE_API} ."
                }
                dir('react-ui') {
                    sh "docker build -t ${IMAGE_UI} ."
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                sh """
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker push ${IMAGE_API}
                    docker push ${IMAGE_UI}
                """
            }
        }

        stage('Package Helm Chart') {
            steps {
                dir('helm-api-ui') {
                    sh """
                        helm lint .
                        helm package . --version ${CHART_VERSION} --app-version ${VERSION}
                    """
                }
            }
        }

        stage('Push Helm Chart to Artifactory') {
            steps {
                dir('helm-api-ui') {
                    sh """
                        curl -u ${ART_USER}:${ART_PASS} \
                        -T ${HELM_CHART_NAME}-${CHART_VERSION}.tgz \
                        ${ARTIFACTORY_HELM_REPO}/${HELM_CHART_NAME}-${CHART_VERSION}.tgz
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Successfully built and pushed Docker images and Helm chart."
        }
        failure {
            echo "Pipeline failed."
        }
    }
}

