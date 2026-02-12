// Jenkinsfile — LMS (api + webapp) -> Docker -> KIND -> Kubernetes
// Works with: Docker, kind, kubectl on Jenkins agent
// Optional: SonarQube scan + Quality Gate

pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '20'))
    disableConcurrentBuilds()
  }

  parameters {
    booleanParam(name: 'FAIL_ON_TEST', defaultValue: false, description: 'Fail pipeline if tests fail')
    booleanParam(name: 'RUN_SONAR', defaultValue: true, description: 'Run SonarQube scan + Quality Gate')
    string(name: 'K8S_NAMESPACE', defaultValue: 'lms', description: 'Kubernetes namespace')
    string(name: 'KIND_CLUSTER', defaultValue: 'kind', description: 'KIND cluster name')
    string(name: 'KUBE_CONTEXT', defaultValue: 'kind-kind', description: 'kubectl context name')
    string(name: 'WEB_NODEPORT', defaultValue: '30080', description: 'NodePort to access web-server')
  }

  environment {
    // Jenkins -> Global Tool Configuration -> NodeJS installation name
    NODEJS_TOOL = 'node20'

    // Jenkins -> Manage Jenkins -> System -> SonarQube servers -> Name
    SONARQUBE_SERVER = 'sonarqube'

    // Docker image names
    API_IMAGE = 'lms-public-api'
    WEB_IMAGE = 'lms-web'
  }

  stages {

    stage('Checkout') {
      steps {
        cleanWs()
        checkout scm

        sh(label: 'versions', script: '''
          set -e
          echo "Node: $(node -v || true)"
          echo "NPM : $(npm -v || true)"
          docker version >/dev/null 2>&1 && echo "Docker OK" || echo "Docker NOT available"
          kubectl version --client=true || true
          kind version || true
        ''')

        sh(label: 'list workspace', script: 'ls -la')
      }
    }

    stage('Tool Install') {
      steps {
        // This injects NODE_HOME and updates PATH for Node
        tool name: "${env.NODEJS_TOOL}", type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation'
      }
    }

    stage('Install Dependencies') {
      steps {
        sh(label: 'npm ci (api + webapp)', script: '''
          set -euo pipefail
          if [ -d "api" ]; then (cd api && npm ci); fi
          if [ -d "webapp" ]; then (cd webapp && npm ci); fi
        ''')
      }
    }

    stage('Build') {
      steps {
        sh(label: 'npm run build', script: '''
          set -euo pipefail
          if [ -d "api" ]; then (cd api && npm run build --if-present); fi
          if [ -d "webapp" ]; then (cd webapp && npm run build --if-present); fi
        ''')
      }
    }

    stage('Test') {
      steps {
        script {
          int status = sh(returnStatus: true, label: 'npm test (api + webapp)', script: '''
            set -euo pipefail
            status=0
            if [ -d "api" ]; then (cd api && npm test --if-present) || status=$?; fi
            if [ -d "webapp" ]; then (cd webapp && npm test --if-present) || status=$?; fi
            exit $status
          ''')

          if (status != 0) {
            echo "⚠️ Tests returned non-zero exit code: ${status}"
            if (params.FAIL_ON_TEST) {
              error("FAIL_ON_TEST=true → failing pipeline due to test failure")
            } else {
              echo "Continuing pipeline (FAIL_ON_TEST=false)"
            }
          }
        }
      }
    }

    stage('SonarQube Scan') {
      when { expression { return params.RUN_SONAR } }
      steps {
        withSonarQubeEnv("${env.SONARQUBE_SERVER}") {
          sh(label: 'sonar-scanner', script: '''
            set -euo pipefail
            # Uses sonar-project.properties in repo
            sonar-scanner
          ''')
        }
      }
    }

    stage('Quality Gate') {
      when { expression { return params.RUN_SONAR } }
      steps {
        timeout(time: 3, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Docker Build + Load into KIND') {
      steps {
        sh(label: 'docker build + kind load', script: '''
          set -euo pipefail
          TAG="${BUILD_NUMBER}"

          docker build -t ${API_IMAGE}:${TAG} api/
          docker build -t ${WEB_IMAGE}:${TAG} webapp/

          echo "KIND clusters:"
          kind get clusters

          kind load docker-image ${API_IMAGE}:${TAG} --name "${KIND_CLUSTER}"
          kind load docker-image ${WEB_IMAGE}:${TAG} --name "${KIND_CLUSTER}"

          echo "Loaded images into KIND cluster: ${KIND_CLUSTER}"
        ''')
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        sh(label: 'kubectl apply + rollout', script: '''
          set -euo pipefail
          NS="${K8S_NAMESPACE}"
          TAG="${BUILD_NUMBER}"

          # Ensure correct context (your issue before)
          kubectl config use-context "${KUBE_CONTEXT}"

          kubectl get ns "${NS}" >/dev/null 2>&1 || kubectl create ns "${NS}"

          # Don't modify repo files permanently
          cp -f lms-api.yaml lms-api.deploy.yaml
          cp -f lms-web.yaml lms-web.deploy.yaml

          sed -i "s/:IMAGE_TAG/:${TAG}/g" lms-api.deploy.yaml
          sed -i "s/:IMAGE_TAG/:${TAG}/g" lms-web.deploy.yaml

          kubectl -n "${NS}" apply -f postgres.yaml
          kubectl -n "${NS}" apply -f lms-api.deploy.yaml
          kubectl -n "${NS}" apply -f lms-web.deploy.yaml

          kubectl -n "${NS}" rollout status statefulset/local-db --timeout=180s || true
          kubectl -n "${NS}" rollout status deploy/api-server --timeout=180s
          kubectl -n "${NS}" rollout status deploy/web-server --timeout=180s

          kubectl -n "${NS}" get pods -o wide
          kubectl -n "${NS}" get svc
        ''')
      }
    }

    stage('App URL (KIND NodePort)') {
      steps {
        sh(label: 'print URL', script: '''
          set -euo pipefail
          kubectl config use-context "${KUBE_CONTEXT}" >/dev/null 2>&1 || true

          # Try to get node IP (KIND usually local)
          NODE_IP="$(ip -4 a | awk '/inet / && $2 !~ /^127/ {print $2}' | head -n1 | cut -d/ -f1 || true)"

          echo "✅ App should be reachable on NodePort:"
          echo "   http://${NODE_IP}:${WEB_NODEPORT}"
          echo ""
          echo "If NODE_IP is empty (WSL/network), use localhost via port-forward:"
          echo "  kubectl -n ${K8S_NAMESPACE} port-forward svc/web-server 8090:80 --address 0.0.0.0"
        ''')
      }
    }
  }

  post {
    always {
      sh(label: 'k8s status (always)', script: '''
        set +e
        kubectl config use-context "${KUBE_CONTEXT}" >/dev/null 2>&1 || true
        kubectl -n "${K8S_NAMESPACE}" get pods -o wide || true
        kubectl -n "${K8S_NAMESPACE}" get svc || true
      ''')
      archiveArtifacts artifacts: '**/lms-*.deploy.yaml', allowEmptyArchive: true
    }

    success {
      echo "✅ Pipeline succeeded."
    }

    failure {
      echo "❌ Pipeline failed. Check stage logs above."
    }
  }
}
