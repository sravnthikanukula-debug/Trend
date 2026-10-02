pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-creds')
        IMAGE_NAME = "sravanthikanukuladocker/trend-app"
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        AWS_REGION = "us-west-2"
        EKS_CLUSTER_NAME = "trend-cluster"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest ."
            }
        }

        stage('Push to DockerHub') {
            steps {
                sh "echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin"
                sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                sh "docker push ${IMAGE_NAME}:latest"
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh "aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}"
                sh "sed -i 's|${IMAGE_NAME}:v1|${IMAGE_NAME}:${IMAGE_TAG}|g' k8s/deployment.yaml"
                sh "kubectl apply -f k8s/deployment.yaml --validate=false"
                sh "kubectl apply -f k8s/service.yaml --validate=false"
                sh "kubectl rollout status deployment/trend-app --timeout=120s"
                sh "kubectl get svc trend-app-svc -o wide"
            }
        }
    }

    post {
        always {
            sh "docker logout"
        }
    }
}
