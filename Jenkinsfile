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
                    sh """
                    ${scannerHome}/bin/sonar-scanner \
                    -Dsonar.projectKey=juice-shop \
                    -Dsonar.sources=. \
                    -Dsonar.javascript.node.maxspace=2048 \
                    -Dsonar.exclusions=**/node_modules/**,**/test/**,**/*.spec.ts,**/*.spec.js,**/frontend/node_modules/**
                    """
                }
            }
        }

        stage('SAST: Quality Gate Check') {
            steps {
                echo "Waiting for SonarQube Quality Gate results..."
                timeout(time: 5, unit: 'MINUTES') {
                    // This command pauses the pipeline until the webhook is received.
                    // If the Quality Gate fails, 'abortPipeline: true' instantly kills the build.
                    waitForQualityGate abortPipeline: true
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
                --severity CRITICAL \
                --exit-code 1 \
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
                APP_IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' juice-shop-app)
                echo "Target IP discovered as: $APP_IP"
                
                docker rm -f zap-scanner || true
               
                docker run --name zap-scanner \
                --user root \
                -v /zap/wrk \
                zaproxy/zap-stable zap-baseline.py \
                -t http://$APP_IP:3000 \
                -r zap_report.html || true 
                
                # Copy the file out of the container
                docker cp zap-scanner:/zap/wrk/zap_report.html ./zap_report.html || echo "ZAP report not found"
                
                # Clean up container and the dummy volume
                docker rm -f -v zap-scanner || true
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