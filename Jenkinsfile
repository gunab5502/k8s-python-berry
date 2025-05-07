
pipeline {
    agent none

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
        ARGOCD_AUTH_TOKEN = credentials('argocd-auth-token')
    }

    stages {

        stage('Checkout Code') {
            agent { label 'docker' }
            steps {
                checkout scm
                sh 'pwd && ls -l'
            }
        }

        stage('Build Docker Images') {
            agent { label 'docker' }
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
            agent { label 'docker' }
            steps {
                sh """
                    printf "%s" "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker push ${IMAGE_API}
                    docker push ${IMAGE_UI}
                """
            }
        }

        stage('Package Helm Chart') {
            agent { label 'k8s' }
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
            agent { label 'k8s' }
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

        stage('Trigger ArgoCD Sync') {
            agent { label 'k8s' }
            steps {
                sh """
                    export ARGOCD_SERVER=a44656053831547038ff313ff47c335d-333561809.us-east-1.elb.amazonaws.com
                    argocd login $ARGOCD_SERVER --insecure --auth-token $ARGOCD_AUTH_TOKEN
                    argocd app sync ${HELM_RELEASE_NAME}
                """
            }
        }
    }

    post {
        success {
            echo "Successfully deployed version ${VERSION} via ArgoCD sync."
        }
        failure {
            echo "Pipeline failed."
        }
    }
}
