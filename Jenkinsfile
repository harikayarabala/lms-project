pipeline {
  agent any

  options { timestamps() }

  environment {
    AWS_REGION  = "ap-south-1"
    AWS_ACCOUNT = "032073356223"
    EKS_CLUSTER = "ekswithlms"
    NAMESPACE   = "lms"

    ECR_REGISTRY  = "${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    API_REPO      = "lms-backend"
    WEB_REPO      = "lms-frontend"

    API_DIR  = "api"
    WEB_DIR  = "webapp"

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

          echo "Repo files:"
          ls -la

          echo "App dirs:"
          ls -la ${API_DIR}
          ls -la ${WEB_DIR}

          echo "Manifests:"
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

    stage("Build & Push API (api/)") {
      steps {
        sh """
          set -e
          docker build -t ${API_REPO}:${IMAGE_TAG} ${API_DIR}
          docker tag ${API_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${API_REPO}:${IMAGE_TAG}
          docker push ${ECR_REGISTRY}/${API_REPO}:${IMAGE_TAG}
        """
      }
    }

    stage("Build & Push WEB (webapp/)") {
      steps {
        sh """
          set -e
          docker build -t ${WEB_REPO}:${IMAGE_TAG} ${WEB_DIR}
          docker tag ${WEB_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${WEB_REPO}:${IMAGE_TAG}
          docker push ${ECR_REGISTRY}/${WEB_REPO}:${IMAGE_TAG}
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

    stage("Verify EBS CSI Driver") {
      steps {
        sh """
          set -e
          echo "Checking EBS CSI pods..."
          kubectl get pods -n kube-system | grep -i ebs || true

          # Fail if controller not running (prevents PVC stuck Pending)
          CNT=\$(kubectl get pods -n kube-system | grep -i ebs-csi-controller | grep Running | wc -l || true)
          if [ "\$CNT" -lt 1 ]; then
            echo "EBS CSI controller not running. Fix aws-ebs-csi-driver add-on/IAM first."
            exit 1
          fi
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

          kubectl get deploy -n ${NAMESPACE}
          kubectl get svc -n ${NAMESPACE}
        """
      }
    }

    stage("Update Images & Rollout") {
      steps {
        sh """
          set -e

          # ✅ Matches your YAML:
          # deployment: api-server, container: api-server
          kubectl set image deployment/api-server \
            api-server=${ECR_REGISTRY}/${API_REPO}:${IMAGE_TAG} \
            -n ${NAMESPACE}

          # ✅ Matches your YAML:
          # deployment: web-server, container: web-server
          kubectl set image deployment/web-server \
            web-server=${ECR_REGISTRY}/${WEB_REPO}:${IMAGE_TAG} \
            -n ${NAMESPACE}

          kubectl rollout status deployment/api-server -n ${NAMESPACE} --timeout=180s
          kubectl rollout status deployment/web-server -n ${NAMESPACE} --timeout=180s

          kubectl get pods -n ${NAMESPACE} -o wide
          kubectl get svc  -n ${NAMESPACE}
        """
      }
    }
  }

  post {
    always {
      sh "docker system prune -af || true"
    }
  }
}
