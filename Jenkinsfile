pipeline {
    agent any

    environment {
        DOCKERHUB_USERNAME = 'swarupsinh10'
        IMAGE_NAME         = 'devops-static-app'
        DOCKER_CREDS_ID    = 'dockerhub-creds'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER} .'
                sh 'docker tag ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER} ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest'
            }
        }

        stage('Docker Hub Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                    sh 'docker push ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER}'
                    sh 'docker push ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest'
                }
            }
        }

        stage('Kubernetes Deploy') {
            steps {
                sh 'sed -i "s|image:.*|image: ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER}|g" k8s-deployment.yaml'
                sh 'kubectl apply -f k8s-deployment.yaml'
                sh 'kubectl apply -f k8s-service.yaml'
                sh 'kubectl rollout status deployment/static-web-deployment'
            }
        }
    }

    post {
        always {
            sh 'docker logout'
        }
    }
}