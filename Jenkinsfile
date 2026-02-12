pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  parameters {
    string(name: 'GIT_URL', defaultValue: 'https://github.com/harikayarabala/lms-project.git', description: 'Git repo URL')
    string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Branch to build')

    // Quality controls
    booleanParam(name: 'FAIL_ON_TEST', defaultValue: false, description: 'If true, pipeline fails when tests fail')
    booleanParam(name: 'FAIL_ON_QUALITY_GATE', defaultValue: false, description: 'If true, pipeline fails when SonarQube Quality Gate fails')

    // Kubernetes deploy (KIND on Ubuntu)
    booleanParam(name: 'DEPLOY_K8S', defaultValue: true, description: 'Deploy to Kubernetes using kubectl (KIND)')
    string(name: 'K8S_NAMESPACE', defaultValue: 'lms', description: 'Kubernetes namespace')
    string(name: 'KIND_CLUSTER', defaultValue: 'kind', description: 'KIND cluster name (kind get clusters)')
  }

  environment {
    // Jenkins -> Manage Jenkins -> System -> SonarQube servers -> Name
    SONARQUBE_SERVER_NAME = "sonarqube"

    // Jenkins -> Global Tool Configuration -> SonarQube Scanner -> Name
    SONAR_SCANNER_TOOL = "sonar-scanner"

    DOCKER_BUILDKIT = "1"
  }

  tools {
    // Jenkins -> Global Tool Configuration -> NodeJS -> Name
    nodejs "node20"
  }

  stages {

    stage('Checkout') {
      steps {
        cleanWs()
        git branch: params.GIT_BRANCH, url: params.GIT_URL

        sh(label: 'versions', script: '''
          bash -lc '
            set -e
            echo "Node: $(node -v)"
            echo "NPM : $(npm -v)"
            docker version >/dev/null 2>&1 && echo "Docker OK" || echo "Docker NOT available"
            kubectl version --client=true || true
            kind version || true
          '
        ''')
        sh(label: 'list workspace', script: 'ls -la')
      }
    }

    stage('Install Dependencies') {
      steps {
        sh(label: 'npm ci (api + webapp)', script: '''
          bash -lc '
            set -euo pipefail

            if [ -d "api" ]; then
              (cd api && npm ci)
            fi

            if [ -d "webapp" ]; then
              (cd webapp && npm ci)
            fi
          '
        ''')
      }
    }

    stage('Build') {
      steps {
        sh(label: 'npm run build', script: '''
          bash -lc '
            set -euo pipefail

            if [ -d "api" ]; then
              (cd api && npm run build --if-present)
            fi

            if [ -d "webapp" ]; then
              (cd webapp && npm run build --if-present)
            fi
          '
        ''')
      }
    }

    stage('Test') {
      steps {
        script {
          def testStatus = sh(
            label: 'npm test (api + webapp)',
            returnStatus: true,
            script: '''
              bash -lc '
                set -euo pipefail
                status=0

                if [ -d "api" ]; then
                  (cd api && npm test --if-present) || status=$?
                fi

                if [ -d "webapp" ]; then
                  (cd webapp && npm test --if-present) || status=$?
                fi

                exit $status
              '
            '''
          )

          if (testStatus != 0) {
            echo "⚠️ Tests returned non-zero exit code: ${testStatus}"
            if (params.FAIL_ON_TEST) {
              error("Failing pipeline because FAIL_ON_TEST=true")
            } else {
              echo "Continuing pipeline (FAIL_ON_TEST=false)"
            }
          } else {
            echo "✅ Tests passed"
          }
        }
      }
    }

    stage('SonarQube Scan') {
      steps {
        script {
          def scannerHome = tool(env.SONAR_SCANNER_TOOL)
          withSonarQubeEnv(env.SONARQUBE_SERVER_NAME) {
            sh(label: 'sonar-scanner', script: """
              bash -lc '
                set -euo pipefail
                ${scannerHome}/bin/sonar-scanner
              '
            """)
          }
        }
      }
    }

    stage('Quality Gate') {
      steps {
        script {
          timeout(time: 20, unit: 'MINUTES') {
            def qg = waitForQualityGate abortPipeline: params.FAIL_ON_QUALITY_GATE
            echo "Quality Gate Status: ${qg.status}"
          }

          if (params.FAIL_ON_QUALITY_GATE == false) {
            echo "Continuing even if Quality Gate is not OK (FAIL_ON_QUALITY_GATE=false)."
          }
        }
      }
    }

    stage('Docker Build + Load into KIND') {
      when { expression { return params.DEPLOY_K8S } }
      steps {
        sh(label: 'docker build + kind load', script: '''
          bash -lc '
            set -euo pipefail

            TAG="${BUILD_NUMBER}"

            # Build images exactly as your K8s YAML expects
            docker build -t lms-public-api:${TAG} api/
            docker build -t lms-web:${TAG} webapp/

            echo "KIND clusters:"
            kind get clusters

            # Load images into the correct KIND cluster
            kind load docker-image lms-public-api:${TAG} --name "${KIND_CLUSTER}"
            kind load docker-image lms-web:${TAG} --name "${KIND_CLUSTER}"

            echo "Loaded images into KIND cluster: ${KIND_CLUSTER}"
          '
        ''')
      }
    }

    stage('Deploy to Kubernetes') {
      when { expression { return params.DEPLOY_K8S } }
      steps {
        sh(label: 'kubectl apply + rollout', script: '''
          bash -lc '
            set -euo pipefail

            NS="${K8S_NAMESPACE}"
            TAG="${BUILD_NUMBER}"

            # Ensure namespace exists
            kubectl get ns "${NS}" >/dev/null 2>&1 || kubectl create ns "${NS}"

            # Do NOT permanently modify repo yaml. Create deploy copies.
            cp -f lms-api.yaml lms-api.deploy.yaml
            cp -f lms-web.yaml lms-web.deploy.yaml

            # Replace placeholder IMAGE_TAG with Jenkins build number
            sed -i "s/:IMAGE_TAG/:${TAG}/g" lms-api.deploy.yaml
            sed -i "s/:IMAGE_TAG/:${TAG}/g" lms-web.deploy.yaml

            # Apply manifests
            kubectl -n "${NS}" apply -f postgres.yaml
            kubectl -n "${NS}" apply -f lms-api.deploy.yaml
            kubectl -n "${NS}" apply -f lms-web.deploy.yaml

            # Rollout checks
            kubectl -n "${NS}" rollout status statefulset/local-db --timeout=180s
            kubectl -n "${NS}" rollout status deploy/api-server --timeout=180s
            kubectl -n "${NS}" rollout status deploy/web-server --timeout=180s

            # Show status
            kubectl -n "${NS}" get pods -o wide
            kubectl -n "${NS}" get svc
          '
        ''')
      }
    }

    stage('App URL (KIND NodePort)') {
      when { expression { return params.DEPLOY_K8S } }
      steps {
        echo "✅ If your web-server service is NodePort 30080, open:"
        echo "   http://<ubuntu-ip>:30080  (or http://localhost:30080 on the Jenkins machine)"
      }
    }
  }

  post {
    always {
      sh(label: 'k8s status (always)', script: '''
        bash -lc '
          set +e
          kubectl -n "${K8S_NAMESPACE}" get pods -o wide || true
          kubectl -n "${K8S_NAMESPACE}" get svc || true
        '
      ''')
    }
    success {
      echo "✅ Pipeline completed successfully."
    }
    failure {
      echo "❌ Pipeline failed. Check stage logs above."
    }
  }
}
