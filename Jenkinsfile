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

    ECR_REGISTRY  = "${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    BACKEND_REPO  = "lms-backend"
    FRONTEND_REPO = "lms-frontend"

    // ✅ repo paths (based on your Jenkins workspace output)
    BACKEND_DIR  = "api"
    FRONTEND_DIR = "webapp"

    // build tag
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
          echo "AWS:"
          aws --version

          echo "kubectl:"
          kubectl version --client=true

          echo "Docker:"
          docker --version

          echo "Workspace files:"
          ls -la

          echo "Repo folders:"
          ls -la ${BACKEND_DIR}
          ls -la ${FRONTEND_DIR}

          echo "YAML files in root:"
          ls -la postgres.yaml lms-api.yaml lms-web.yaml
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

    stage("Build & Push Backend (api)") {
      steps {
        sh """
          set -e
          docker build -t ${BACKEND_REPO}:${IMAGE_TAG} ${BACKEND_DIR}
          docker tag ${BACKEND_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${BACKEND_REPO}:${IMAGE_TAG}
          docker push ${ECR_REGISTRY}/${BACKEND_REPO}:${IMAGE_TAG}
        """
      }
    }

    stage("Build & Push Frontend (webapp)") {
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
          kubectl apply -n ${NAMESPACE} -f postgres.yaml
          kubectl apply -n ${NAMESPACE} -f lms-api.yaml
          kubectl apply -n ${NAMESPACE} -f lms-web.yaml

          kubectl get all -n ${NAMESPACE}
        """
      }
    }

    stage("Update Images & Rollout") {
      steps {
        sh """
          set -e

          # ============================================================
          # IMPORTANT: Update deployment + container names if different.
          # Use:
          # kubectl get deploy -n ${NAMESPACE}
          # kubectl get deploy <name> -n ${NAMESPACE} -o jsonpath='{.spec.template.spec.containers[*].name}'
          # ============================================================

          # ✅ Assumption: deployment name = lms-api, container name = lms-api
          kubectl set image deployment/lms-api lms-api=${ECR_REGISTRY}/${BACKEND_REPO}:${IMAGE_TAG} -n ${NAMESPACE}

          # ✅ Assumption: deployment name = lms-web, container name = lms-web
          kubectl set image deployment/lms-web lms-web=${ECR_REGISTRY}/${FRONTEND_REPO}:${IMAGE_TAG} -n ${NAMESPACE}

          kubectl rollout status deployment/lms-api -n ${NAMESPACE}
          kubectl rollout status deployment/lms-web -n ${NAMESPACE}

          kubectl get pods -n ${NAMESPACE} -o wide
          kubectl get svc -n ${NAMESPACE}
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
