pipeline {
  agent any

  options {
    timestamps()
  }

  environment {
    AWS_REGION   = "ap-south-1"
    AWS_ACCOUNT  = "032073356223"
    EKS_CLUSTER  = "ekswithlms"
    NAMESPACE    = "lms"

    ECR_REGISTRY   = "${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    BACKEND_REPO   = "lms-backend"
    FRONTEND_REPO  = "lms-frontend"

    // Update these paths to match your repo
    BACKEND_DIR = "backend"
    FRONTEND_DIR = "frontend"
    K8S_DIR = "k8s"

    IMAGE_TAG = "${BUILD_NUMBER}"
  }

  stages {

    stage("Checkout") {
      steps {
        git branch: "main", url: "https://github.com/harikayarabala/lms-project.git"
      }
    }

    stage("Prechecks") {
      steps {
        sh """
          set -e
          aws --version
          kubectl version --client=true
          docker --version

          echo "Workspace:"
          ls -la

          echo "Checking expected folders (update Jenkinsfile if different):"
          ls -la ${BACKEND_DIR} || true
          ls -la ${FRONTEND_DIR} || true
          ls -la ${K8S_DIR} || true
        """
      }
    }

    stage("Login to ECR") {
      steps {
        sh """
          set -e
          aws ecr get-login-password --region ${AWS_REGION} \
            | docker login --username AWS --password-stdin ${ECR_REGISTRY}
        """
      }
    }

    stage("Build & Push Backend") {
      steps {
        sh """
          set -e
          docker build -t ${BACKEND_REPO}:${IMAGE_TAG} ${BACKEND_DIR}
          docker tag ${BACKEND_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${BACKEND_REPO}:${IMAGE_TAG}
          docker push ${ECR_REGISTRY}/${BACKEND_REPO}:${IMAGE_TAG}
        """
      }
    }

    stage("Build & Push Frontend") {
      steps {
        sh """
          set -e
          docker build -t ${FRONTEND_REPO}:${IMAGE_TAG} ${FRONTEND_DIR}
          docker tag ${FRONTEND_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${FRONTEND_REPO}:${IMAGE_TAG}
          docker push ${ECR_REGISTRY}/${FRONTEND_REPO}:${IMAGE_TAG}
        """
      }
    }

    stage("Configure kubeconfig") {
      steps {
        sh """
          set -e
          aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}
          kubectl get nodes
          kubectl get ns ${NAMESPACE} || kubectl create ns ${NAMESPACE}
        """
      }
    }

    stage("Apply Manifests") {
      steps {
        sh """
          set -e
          kubectl apply -n ${NAMESPACE} -f ${K8S_DIR}
          kubectl get all -n ${NAMESPACE}
        """
      }
    }

    stage("Update Images & Rollout") {
      steps {
        sh """
          set -e

          # ✅ CHANGE THESE NAMES to match your Kubernetes manifests:
          # deployment/<DEPLOYMENT_NAME> and container name inside that deployment
          kubectl set image deployment/lms-backend  lms-backend=${ECR_REGISTRY}/${BACKEND_REPO}:${IMAGE_TAG}   -n ${NAMESPACE}
          kubectl set image deployment/lms-frontend lms-frontend=${ECR_REGISTRY}/${FRONTEND_REPO}:${IMAGE_TAG} -n ${NAMESPACE}

          kubectl rollout status deployment/lms-backend  -n ${NAMESPACE}
          kubectl rollout status deployment/lms-frontend -n ${NAMESPACE}

          kubectl get pods -n ${NAMESPACE} -o wide
          kubectl get svc  -n ${NAMESPACE}
        """
      }
    }
  }

  post {
    always {
      sh """
        docker system prune -af || true
      """
    }
  }
}
