pipeline {
    agent any
    
    environment {
        // AWS & ECR Configuration
        AWS_ACCOUNT_ID = '017161968499' 
        AWS_REGION = 'ap-south-1'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_BACKEND_REPO = 'nestnet-backend'
        ECR_FRONTEND_REPO = 'nestnet-frontend'
        
        // EKS Configuration (REPLACE WITH YOUR ACTUAL CLUSTER NAME)
        EKS_CLUSTER_NAME = 'nestnet-cluster'
        
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Start Ephemeral SonarQube') {
            steps {
                sh '''
                    docker rm -f sonarqube-ci || true
                    docker run -d --name sonarqube-ci -p 9000:9000 sonarqube:9.9.5-community
                    
                    echo "Waiting for SonarQube to initialize..."
                    until curl -s http://localhost:9000/api/system/status | grep -q '"status":"UP"'; do
                        sleep 5
                        echo "Still waiting..."
                    done
                    echo "SonarQube is UP!"
                    
                    SQ_RESPONSE=$(curl -s -u admin:admin -X POST "http://localhost:9000/api/user_tokens/generate" -d "name=ci-pipeline")
                    SQ_TOKEN=$(echo "$SQ_RESPONSE" | jq -r '.token' | tr -d '\\n\\r\\t ')
                    
                    if [ "$SQ_TOKEN" = "null" ] || [ -z "$SQ_TOKEN" ] || [ ${#SQ_TOKEN} -lt 10 ]; then
                        echo "ERROR: Failed to generate token. SonarQube response: $SQ_RESPONSE"
                        exit 1
                    fi
                    
                    echo "$SQ_TOKEN" > sq_token.txt
                    echo "Token generated successfully."
                '''
            }
        }

        stage('Source SAST & SCA') {
            steps {
                script {
                    def sqToken = sh(script: 'cat sq_token.txt | tr -d "\\n\\r\\t "', returnStdout: true).trim()
                    echo "SonarQube Token extracted (Length: ${sqToken.length()}): ${sqToken.take(10)}..."
                    
                    sh """
                        docker run --rm \\
                          --network host \\
                          -v \$(pwd):/usr/src \\
                          sonarsource/sonar-scanner-cli:latest \\
                          -Dsonar.projectKey=NestNet \\
                          -Dsonar.sources=backend,frontend \\
                          -Dsonar.host.url=http://localhost:9000 \\
                          -Dsonar.token=${sqToken} \\
                          -Dsonar.login=${sqToken}
                    """
                }
                
                dir('backend') { sh 'npm audit --audit-level=high || true' }
                dir('frontend') { sh 'npm audit --audit-level=high || true' }
            }
        }

        stage('Teardown Ephemeral SonarQube') {
            steps {
                sh '''
                    echo "Stopping and removing SonarQube container..."
                    docker stop sonarqube-ci || true
                    docker rm sonarqube-ci || true
                    rm -f sq_token.txt || true
                '''
            }
        }

        stage('Docker Build') {
            parallel {
                stage('Build Backend') {
                    steps {
                        sh "docker build -t ${ECR_REGISTRY}/${ECR_BACKEND_REPO}:${IMAGE_TAG} -f backend/Dockerfile backend"
                    }
                }
                stage('Build Frontend') {
                    steps {
                        sh "docker build -t ${ECR_REGISTRY}/${ECR_FRONTEND_REPO}:${IMAGE_TAG} -f frontend/Dockerfile frontend"
                    }
                }
            }
        }

        // --- FIXED: Trivy now runs inside Docker ---
        stage('Container SAST (Trivy)') {
            steps {
                script {
                    echo "Scanning Backend Image with Trivy..."
                    sh """
                        docker run --rm \\
                          -v /var/run/docker.sock:/var/run/docker.sock \\
                          aquasec/trivy:latest image \\
                          --exit-code 1 \\
                          --severity HIGH,CRITICAL \\
                          ${ECR_REGISTRY}/${ECR_BACKEND_REPO}:${IMAGE_TAG}
                    """
                    
                    echo "Scanning Frontend Image with Trivy..."
                    sh """
                        docker run --rm \\
                          -v /var/run/docker.sock:/var/run/docker.sock \\
                          aquasec/trivy:latest image \\
                          --exit-code 1 \\
                          --severity HIGH,CRITICAL \\
                          ${ECR_REGISTRY}/${ECR_FRONTEND_REPO}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Local DAST (OWASP ZAP)') {
            steps {
                script {
                    sh '''
                        docker network create nestnet-dast-net || true
                        docker run -d --name dast-backend --network nestnet-dast-net ${ECR_REGISTRY}/${ECR_BACKEND_REPO}:${IMAGE_TAG}
                        sleep 10 
                        docker run -d --name dast-frontend --network nestnet-dast-net -p 8080:80 ${ECR_REGISTRY}/${ECR_FRONTEND_REPO}:${IMAGE_TAG}
                        sleep 10
                    '''
                    
                    sh '''
                        docker run --rm --network host -v $(pwd):/zap/wrk/ -t owasp/zap2docker-stable zap-baseline.py \\
                        -t http://localhost:8080 -r zap_report.html || true
                    '''
                    
                    sh '''
                        docker stop dast-backend dast-frontend || true
                        docker rm dast-backend dast-frontend || true
                        docker network rm nestnet-dast-net || true
                    '''
                }
            }
        }

        stage('Push to AWS ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    docker push ${ECR_REGISTRY}/${ECR_BACKEND_REPO}:${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${ECR_FRONTEND_REPO}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}
                    
                    kubectl apply -f k8s/namespace.yaml
                    kubectl apply -f k8s/backend.yaml
                    kubectl apply -f k8s/frontend.yaml
                    kubectl apply -f k8s/ingress.yaml
                    
                    kubectl set image deployment/nestnet-backend nestnet-backend=${ECR_REGISTRY}/${ECR_BACKEND_REPO}:${IMAGE_TAG} -n production
                    kubectl set image deployment/nestnet-frontend nestnet-frontend=${ECR_REGISTRY}/${ECR_FRONTEND_REPO}:${IMAGE_TAG} -n production
                    
                    kubectl rollout status deployment/nestnet-backend -n production
                    kubectl rollout status deployment/nestnet-frontend -n production
                '''
            }
        }
    }
}
