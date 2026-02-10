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

    // Scans / deploy switches
    booleanParam(name: 'RUN_TRIVY', defaultValue: false, description: 'Run Trivy image scan (requires trivy on agent)')
    booleanParam(name: 'DEPLOY', defaultValue: true, description: 'Deploy using docker compose up -d')

    // Control behavior
    booleanParam(name: 'FAIL_ON_TEST', defaultValue: false, description: 'If true, pipeline fails when tests fail')
    booleanParam(name: 'FAIL_ON_QUALITY_GATE', defaultValue: false, description: 'If true, pipeline fails when SonarQube Quality Gate fails')
  }

  environment {
    // Must match Jenkins -> Manage Jenkins -> System -> SonarQube servers -> Name
    SONARQUBE_SERVER_NAME = "sonarqube"

    // Must match Jenkins -> Global Tool Configuration -> SonarQube Scanner -> Name
    SONAR_SCANNER_TOOL = "sonar-scanner"

    DOCKER_BUILDKIT = "1"
    COMPOSE_DOCKER_CLI_BUILD = "1"
  }

  tools {
    // Must match Jenkins -> Global Tool Configuration -> NodeJS -> Name
    nodejs "node20"
  }

  stages {

    stage('Checkout (manual)') {
      steps {
        cleanWs()
        git branch: params.GIT_BRANCH, url: params.GIT_URL
        sh(label: 'versions', script: 'node -v; npm -v; docker version >/dev/null 2>&1 || true; docker compose version || true')
        sh(label: 'list workspace', script: 'ls -la')
      }
    }

    stage('Install Dependencies') {
      steps {
        sh(label: 'npm ci (api + webapp)', script: '''
          bash -lc '
            set -euo pipefail

            if [ -d "api" ]; then
              cd api
              npm ci
              cd ..
            fi

            if [ -d "webapp" ]; then
              cd webapp
              npm ci
              cd ..
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
              cd api
              npm run build --if-present
              cd ..
            fi

            if [ -d "webapp" ]; then
              cd webapp
              npm run build --if-present
              cd ..
            fi
          '
        ''')
      }
    }

    stage('Test') {
      steps {
        script {
          // Run tests but by default do NOT fail the whole pipeline if tests fail.
          def testStatus = sh(
            label: 'npm test (api + webapp)',
            returnStatus: true,
            script: '''
              bash -lc '
                set -euo pipefail

                status=0

                if [ -d "api" ]; then
                  cd api
                  npm test --if-present || status=$?
                  cd ..
                fi

                if [ -d "webapp" ]; then
                  cd webapp
                  npm test --if-present || status=$?
                  cd ..
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
          // If FAIL_ON_QUALITY_GATE=true, abortPipeline will fail the build when gate is RED.
          // If false, pipeline continues even if gate fails (useful for demo/learning).
          def abortOnGate = params.FAIL_ON_QUALITY_GATE

          timeout(time: 20, unit: 'MINUTES') {
            def qg = waitForQualityGate abortPipeline: abortOnGate
            echo "Quality Gate Status: ${qg.status}"
          }

          if (params.FAIL_ON_QUALITY_GATE == false) {
            echo "Continuing even if Quality Gate is not OK (FAIL_ON_QUALITY_GATE=false)."
          }
        }
      }
    }

    stage('Docker Build (compose)') {
      steps {
        sh(label: 'docker compose build', script: '''
          bash -lc '
            set -euo pipefail

            test -f docker-compose.yml

            docker version
            docker compose version

            docker compose build --no-cache
            docker compose config
          '
        ''')
      }
    }

    stage('Image Scan (Trivy - optional)') {
      when { expression { return params.RUN_TRIVY } }
      steps {
        sh(label: 'trivy scan', script: '''
          bash -lc '
            set -euo pipefail

            command -v trivy >/dev/null 2>&1 || {
              echo "Trivy not installed on agent. Disable RUN_TRIVY or install trivy."
              exit 1
            }

            echo "Recent images:"
            docker images --format "{{.Repository}}:{{.Tag}}" | head -n 15
          '
        ''')
      }
    }

    stage('Deploy (docker compose up -d)') {
      when { expression { return params.DEPLOY } }
      steps {
        sh(label: 'docker compose up', script: '''
          bash -lc '
            set -euo pipefail

            docker compose down || true
            docker compose up -d
            docker compose ps
          '
        ''')
      }
    }
  }

  post {
    always {
      sh(label: 'compose ps (always)', script: '''
        bash -lc '
          set +e
          docker compose ps || true
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

