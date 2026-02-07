pipeline {
    agent any
    stages {
        stage('Code Analysis') {
            steps {
               echo 'Sonar Analysis' 
               sh 'sudo docker run --rm -e SONAR_HOST_URL="http://18.191.239.141:9000" -v ".:/usr/src" sonarsource/sonar-scanner-cli -Dsonar.token="sqp_4a0e378bddba5dad2621e3e8e63d4e785b2820bc" -Dsonar.projectKey=lms'
            }
        }
        stage('Build - dev') {
            steps {
                echo 'Building Project' 
                //sh 'cd webapp && npm install && npm run build'
            }
        }
        stage('Build - qa') {
            steps {
                echo 'Building Project On Docker' 
                sh 'cd webapp && sudo docker build -t ravi2krishna/lms-fe .'
            }
        }
        stage('Deploy Frontend - dev') {
            steps {
                echo 'Deploying Project'
                sh ' sudo docker container rm -f web'
                sh 'sudo docker container run -dt --name web -p 8081:80 ravi2krishna/lms-fe'
                // sh 'sudo rm -rf /var/www/html/*' 
                // sh 'sudo cp -r webapp/dist/* /var/www/html'
            }
        }
        stage('Deploy Frontend - k8s') {
            steps {
                echo 'Deploying Project On Kubernetes'
                sh 'kubectl' 
            }
        }
        stage('Cleanup Workspace') {
            steps {
                echo 'Cleaning Up Old Resources'
                cleanWs()
            }
        }

    }
}