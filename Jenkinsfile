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
                # 1. Fetch the internal Docker IP of the Juice Shop container
                APP_IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' juice-shop-app)
                echo "Target IP discovered as: $APP_IP"
                
                # Clean up any leftover zap containers from previous failed runs
                docker rm -f zap-scanner || true
                
                # 2. Run ZAP (Notice we removed the -v volume mount and --rm flag)
                # We name the container 'zap-scanner' so we can reference it in the next step.
                docker run --name zap-scanner \
                owasp/zap2docker-stable zap-baseline.py \
                -t http://$APP_IP:3000 \
                -r zap_report.html || true 
                
                # 3. Manually copy the generated report out of the ZAP container into the Jenkins workspace
                docker cp zap-scanner:/zap/wrk/zap_report.html ./zap_report.html || echo "ZAP report not found"
                
                # 4. Delete the ZAP container now that we have our file
                docker rm -f zap-scanner || true
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