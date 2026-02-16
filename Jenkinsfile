pipeline {
    agent any

    environment {
        AWS_REGION = "ap-south-1"
        CLUSTER_NAME = "ekswithavinash"
        ACCOUNT_ID = sh(script: "aws sts get-caller-identity --query Account --output text", returnStdout: true).trim()
        ECR = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify Tools') {
            steps {
                sh '''
                set -e
                aws --version
                kubectl version --client
                docker version
                node -v
                npm -v
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                aws ecr get-login-password --region $AWS_REGION | \
                docker login --username AWS --password-stdin $ECR
                '''
            }
        }

        stage('Build Images') {
            steps {
                sh '''
                docker build -t lms-api ./api
                docker build -t lms-web ./webapp
                '''
            }
        }

        stage('Tag Images') {
            steps {
                sh '''
                docker tag lms-api:latest $ECR/lms-api:latest
                docker tag lms-web:latest $ECR/lms-web:latest
                '''
            }
        }

        stage('Push Images') {
            steps {
                sh '''
                docker push $ECR/lms-api:latest
                docker push $ECR/lms-web:latest
                '''
            }
        }

        stage('Update Kubeconfig') {
            steps {
                sh '''
                aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                kubectl apply -f postgres.yaml
                kubectl apply -f lms-api.yaml
                kubectl apply -f lms-web.yaml
                '''
            }
        }
    }
}
