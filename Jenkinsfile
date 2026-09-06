pipeline {
    agent any

    environment {
        APP_IP = '172.31.39.180'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh '''
                    python3 -m venv venv
                    ./venv/bin/pip install --upgrade pip
                    ./venv/bin/pip install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    ./venv/bin/pytest -v --cov=. --cov-report=xml
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    withEnv(["PATH+SONAR=${tool 'SonarScanner'}/bin"]) {
                        sh '''
                            sonar-scanner \
                              -Dsonar.projectKey=hello-python \
                              -Dsonar.projectName=hello-python \
                              -Dsonar.sources=app.py \
                              -Dsonar.tests=test_app.py \
                              -Dsonar.python.version=3 \
                              -Dsonar.python.coverage.reportPaths=coverage.xml \
                              -Dsonar.exclusions=venv/**,coverage.xml
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Deploy') {
            steps {
                sshagent(credentials: ['aws-app-ssh']) {
                    sh '''
                        ssh -o ConnectTimeout=10 -o StrictHostKeyChecking=no ubuntu@$APP_IP \
                          "mkdir -p ~/app"
        
                        scp -o ConnectTimeout=10 -o StrictHostKeyChecking=no \
                          app.py requirements.txt \
                          ubuntu@$APP_IP:~/app/
        
                        ssh -o ConnectTimeout=10 -o StrictHostKeyChecking=no ubuntu@$APP_IP \
                          "cd ~/app && python3 -m venv venv && ./venv/bin/pip install -r requirements.txt"
        
                        ssh -o ConnectTimeout=10 -o StrictHostKeyChecking=no ubuntu@$APP_IP \
                          "if [ -f ~/app/app.pid ]; then kill \\$(cat ~/app/app.pid) 2>/dev/null || true; rm -f ~/app/app.pid; fi"
        
                        ssh -o ConnectTimeout=10 -o StrictHostKeyChecking=no ubuntu@$APP_IP \
                          "cd ~/app && nohup ./venv/bin/python app.py </dev/null >app.log 2>&1 & echo \\$! > app.pid"
        
                        sleep 3
        
                        ssh -o ConnectTimeout=10 -o StrictHostKeyChecking=no ubuntu@$APP_IP \
                          "curl -f http://localhost:8080/health"
                    '''
                }
            }
        }
    }
}
