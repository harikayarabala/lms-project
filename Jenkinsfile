pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  environment {
    AWS_REGION  = "ap-south-1"
    EKS_CLUSTER = "ekswithavinash"
    NAMESPACE   = "lms"

    API_REPO    = "lms-public-api"
    WEB_REPO    = "lms-web"

    TAG         = "${BUILD_NUMBER}"
    // Helps node builds on smaller machines
    NODE_OPTIONS = "--max-old-space-size=768"
  }

  stages {

    stage('Checkout') {
      steps {
        cleanWs()
        checkout scm
      }
    }

    stage('Verify Tools') {
      steps {
        sh '''
          set -e
          aws --version
          kubectl version --client=true
          docker version
          node -v
          npm -v
        '''
      }
    }

    stage('Install Deps') {
      steps {
        sh '''
          set -euo pipefail
          (cd api && npm ci --no-audit --no-fund)
          (cd webapp && npm ci --no-audit --no-fund)
        '''
      }
    }

    stage('Build App') {
      steps {
        sh '''
          set -euo pipefail
          (cd api && npm run build --if-present)
          (cd webapp && npm run build --if-present)
        '''
      }
    }

    stage('Build Docker Images') {
      steps {
        sh '''
          set -euo pipefail
          docker build -t ${API_REPO}:${TAG} api
          docker build -t ${WEB_REPO}:${TAG} webapp
        '''
      }
    }

    stage('Login & Push to ECR') {
      steps {
        sh '''
          set -euo pipefail

          ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
          ECR="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

          aws ecr describe-repositories --repository-names ${API_REPO} --region ${AWS_REGION} >/dev/null 2>&1 \
            || aws ecr create-repository --repository-name ${API_REPO} --region ${AWS_REGION} >/dev/null

          aws ecr describe-repositories --repository-names ${WEB_REPO} --region ${AWS_REGION} >/dev/null 2>&1 \
            || aws ecr create-repository --repository-name ${WEB_REPO} --region ${AWS_REGION} >/dev/null

          aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR}

          docker tag ${API_REPO}:${TAG} ${ECR}/${API_REPO}:${TAG}
          docker tag ${WEB_REPO}:${TAG} ${ECR}/${WEB_REPO}:${TAG}

          docker push ${ECR}/${API_REPO}:${TAG}
          docker push ${ECR}/${WEB_REPO}:${TAG}

          echo "${ECR}" > .ecr_uri
        '''
        stash name: "ecr_uri", includes: ".ecr_uri"
      }
    }

    stage('Connect to EKS') {
      steps {
        sh '''
          set -euo pipefail
          aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}
          kubectl get nodes
        '''
      }
    }

    stage('Deploy to EKS') {
      steps {
        unstash "ecr_uri"
        sh '''
          set -euo pipefail

          ECR=$(cat .ecr_uri)

          kubectl get ns ${NAMESPACE} >/dev/null 2>&1 || kubectl create ns ${NAMESPACE}

          kubectl -n ${NAMESPACE} apply -f postgres.yaml
          kubectl -n ${NAMESPACE} apply -f lms-api.yaml
          kubectl -n ${NAMESPACE} apply -f lms-web.yaml

          # Update images (must match your deployment & container names)
          kubectl -n ${NAMESPACE} set image deploy/api-server api-server=${ECR}/${API_REPO}:${TAG} --record
          kubectl -n ${NAMESPACE} set image deploy/web-server web-server=${ECR}/${WEB_REPO}:${TAG} --record

          kubectl -n ${NAMESPACE} rollout status deploy/api-server --timeout=300s
          kubectl -n ${NAMESPACE} rollout status deploy/web-server --timeout=300s

          kubectl -n ${NAMESPACE} get pods -o wide
          kubectl -n ${NAMESPACE} get svc
        '''
      }
    }
  }

  post {
    always {
      sh '''
        set +e
        kubectl -n ${NAMESPACE} get pods -o wide || true
        kubectl -n ${NAMESPACE} get svc || true
      '''
    }
  }
}
