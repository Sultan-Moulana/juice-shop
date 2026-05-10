pipeline {
    agent any
    
    environment {
        IMAGE_NAME = "juice-shop-devsecops"
    }

    stages {
        stage('SAST: SonarQube Analysis') {
            environment {
                scannerHome = tool 'sonar-scanner' 
            }
            tools {
                nodejs 'node'
            }
            steps {
                withSonarQubeEnv('sonarqube') { 
                    // Added the maxspace flag at the end so it won't crash
                    sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=juice-shop -Dsonar.sources=. -Dsonar.javascript.node.maxspace=4096"
                }
            }
        }
        
        stage('Build Target Image') {
            steps {
                echo "Building the vulnerable application..."
                sh 'docker build -t ${IMAGE_NAME}:latest .'
            }
        }
        
        stage('SCA: Trivy Image Scan') {
            steps {
                echo "Scanning Docker image for vulnerabilities..."
                sh '''
                docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                aquasec/trivy image \
                --severity HIGH,CRITICAL \
                --format table \
                ${IMAGE_NAME}:latest
                '''
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                echo "Deploying container for dynamic testing..."
                sh 'docker stop juice-shop-app || true'
                sh 'docker rm juice-shop-app || true'
                sh 'docker run -d --name juice-shop-app -p 3000:3000 ${IMAGE_NAME}:latest'
                sleep time: 15, unit: 'SECONDS'
            }
        }
        
        stage('DAST: OWASP ZAP Scan') {
            steps {
                echo "Running Dynamic Analysis against the live container..."
                sh '''
                docker run --rm -v $(pwd):/zap/wrk/:rw \
                --network="host" \
                owasp/zap2docker-stable zap-baseline.py \
                -t http://localhost:3000 \
                -r zap_report.html || true 
                '''
            }
        }
    }
    
    post {
        always {
            echo "Tearing down staging environment..."
            sh 'docker stop juice-shop-app || true'
            sh 'docker rm juice-shop-app || true'
            echo "Archiving ZAP Security Report..."
            archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
        }
    }
}