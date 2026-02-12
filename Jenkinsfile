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

    booleanParam(name: 'DEPLOY', defaultValue: true, description: 'Deploy to Kubernetes (KIND)')
    booleanParam(name: 'FAIL_ON_TEST', defaultValue: false, description: 'Fail pipeline if tests fail')

    booleanParam(name: 'RUN_SONAR', defaultValue: false, description: 'Run SonarQube scan')
    booleanParam(name: 'FAIL_ON_QUALITY_GATE', defaultValue: false, description: 'Fail pipeline if Quality Gate fails')
  }

  environment {
    // Kubernetes
    KUBE_CONTEXT   = "kind-kind"
    K8S_NAMESPACE  = "lms"
    KIND_CLUSTER   = "kind"

    // Sonar (only used if RUN_SONAR=true)
    SONARQUBE_SERVER_NAME = "sonarqube"
    SONAR_SCANNER_TOOL    = "sonar-scanner"

    DOCKER_BUILDKIT = "1"
    COMPOSE_DOCKER_CLI_BUILD = "1"
  }

  tools {
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
            node -v
            npm -v
            docker version >/dev/null 2>&1 && echo "Docker OK" || echo "Docker NOT available"
            kubectl version --client=true || true
            kind version || true
          '
        ''')
      }
    }

    stage('Install Dependencies') {
      steps {
        sh(label: 'npm ci (api + webapp)', script: '''
          bash -lc '
            set -euo pipefail
            if [ -d "api" ]; then (cd api && npm ci); fi
            if [ -d "webapp" ]; then (cd webapp && npm ci); fi
          '
        ''')
      }
    }

    stage('Build') {
      steps {
        sh(label: 'npm run build', script: '''
          bash -lc '
            set -euo pipefail
            if [ -d "api" ]; then (cd api && npm run build --if-present); fi
            if [ -d "webapp" ]; then (cd webapp && npm run build --if-present); fi
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
                if [ -d "api" ]; then (cd api && npm test --if-present) || status=$?; fi
                if [ -d "webapp" ]; then (cd webapp && npm test --if-present) || status=$?; fi
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
      when { expression { return params.RUN_SONAR } }
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
      when { expression { return params.RUN_SONAR } }
      steps {
        script {
          def abortOnGate = params.FAIL_ON_QUALITY_GATE
          timeout(time: 20, unit: 'MINUTES') {
            def qg = waitForQualityGate abortPipeline: abortOnGate
            echo "Quality Gate Status: ${qg.status}"
          }
          if (!params.FAIL_ON_QUALITY_GATE) {
            echo "Continuing even if Quality Gate is not OK (FAIL_ON_QUALITY_GATE=false)."
          }
        }
      }
    }

    stage('Docker Build + Load into KIND') {
      when { expression { return params.DEPLOY } }
      steps {
        sh(label: 'docker build + kind load', script: '''
          bash -lc '
            set -euo pipefail
            TAG="${BUILD_NUMBER}"

            docker build -t lms-public-api:${TAG} api/
            docker build -t lms-web:${TAG} webapp/

            kind get clusters
            kind load docker-image lms-public-api:${TAG} --name "${KIND_CLUSTER}"
            kind load docker-image lms-web:${TAG} --name "${KIND_CLUSTER}"

            echo "Loaded images into KIND cluster: ${KIND_CLUSTER}"
          '
        ''')
      }
    }

    stage('Deploy to Kubernetes') {
      when { expression { return params.DEPLOY } }
      steps {
        sh(label: 'kubectl apply + rollout', script: '''
          bash -lc '
            set -euo pipefail
            NS="${K8S_NAMESPACE}"
            TAG="${BUILD_NUMBER}"

            # Use correct context for Jenkins user
            kubectl config use-context "${KUBE_CONTEXT}"

            kubectl get ns "${NS}" >/dev/null 2>&1 || kubectl create ns "${NS}"

            # create temp deploy yamls (don’t modify repo files)
            cp -f lms-api.yaml lms-api.deploy.yaml
            cp -f lms-web.yaml lms-web.deploy.yaml

            sed -i "s/:IMAGE_TAG/:${TAG}/g" lms-api.deploy.yaml
            sed -i "s/:IMAGE_TAG/:${TAG}/g" lms-web.deploy.yaml

            kubectl -n "${NS}" apply -f postgres.yaml
            kubectl -n "${NS}" apply -f lms-api.deploy.yaml
            kubectl -n "${NS}" apply -f lms-web.deploy.yaml

            kubectl -n "${NS}" rollout status statefulset/local-db --timeout=180s
            kubectl -n "${NS}" rollout status deploy/api-server --timeout=180s
            kubectl -n "${NS}" rollout status deploy/web-server --timeout=180s

            kubectl -n "${NS}" get pods -o wide
            kubectl -n "${NS}" get svc
          '
        ''')
      }
    }

    stage('App URL (NodePort)') {
      when { expression { return params.DEPLOY } }
      steps {
        sh(label: 'show url', script: '''
          bash -lc '
            set -euo pipefail
            NODE_IP="$(hostname -I | awk "{print \\$1}")"
            echo "Open: http://${NODE_IP}:30080"
          '
        ''')
      }
    }
  }

  post {
    always {
      sh(label: 'k8s status (always)', script: '''
        bash -lc '
          set +e
          kubectl config use-context "${KUBE_CONTEXT}" >/dev/null 2>&1 || true
          kubectl -n "${K8S_NAMESPACE}" get pods -o wide || true
          kubectl -n "${K8S_NAMESPACE}" get svc || true
        '
      ''')
    }
    success { echo "✅ Pipeline completed successfully." }
    failure { echo "❌ Pipeline failed. Check stage logs above." }
  }
}
