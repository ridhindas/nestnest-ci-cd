pipeline {
    agent any
    
    environment {
        // AWS & ECR Configuration (REPLACE THESE VALUES)
        AWS_ACCOUNT_ID = 'i017161968499' 
        AWS_REGION = 'ap-south-1'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_BACKEND_REPO = 'nestnet-backend'
        ECR_FRONTEND_REPO = 'nestnet-frontend'
        
        // EKS Configuration (REPLACE WITH YOUR CLUSTER NAME)
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
                    
                    SQ_TOKEN=$(curl -s -u admin:admin -X POST "http://localhost:9000/api/user_tokens/generate" -d "name=ci-pipeline" | jq -r '.token' | tr -d '\\n\\r')
                    echo "$SQ_TOKEN" > sq_token.txt
                '''
            }
        }

        stage('Source SAST & SCA') {
            steps {
                script {
                    // Read and strictly trim the token
                    def sqToken = sh(
                        script: 'cat sq_token.txt | tr -d "\\n\\r"',
                        returnStdout: true
                    ).trim()
                    
                    echo "SonarQube Token extracted (Length: ${sqToken.length()}): ${sqToken.take(10)}..."
                    
                    // Fail fast if token generation silently failed (e.g., returned "null")
                    if (sqToken == "null" || sqToken.isEmpty()) {
                        error("Failed to generate SonarQube token. The API may have rejected 'admin:admin'. Check SonarQube logs.")
                    }
                    
                    // Run sonar-scanner via Docker with pinned stable version and dual auth flags
                    sh """
                        docker run --rm \\
                          --network host \\
                          -v \$(pwd):/usr/src \\
                          sonarsource/sonar-scanner-cli:6.2.1 \\
                          -Dsonar.projectKey=NestNet \\
                          -Dsonar.sources=backend,frontend \\
                          -Dsonar.host.url=http://localhost:9000 \\
                          -Dsonar.token=${sqToken} \\
                          -Dsonar.login=${sqToken} \\
                          -Dsonar.verbose=true
                    """
                }
                
                // Software Composition Analysis
                dir('backend') { sh 'npm audit --audit-level=high || true' }
                dir('frontend') { sh 'npm audit --audit-level=high || true' }
            }
        }

        stage('Teardown Ephemeral SonarQube') {
            steps {
                sh '''
                    echo "Stopping and removing SonarQube container..."
                    docker stop sonarqube-ci
                    docker rm sonarqube-ci
                    rm -f sq_token.txt
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

        stage('Container SAST (Trivy)') {
            steps {
                sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${ECR_REGISTRY}/${ECR_BACKEND_REPO}:${IMAGE_TAG}"
                sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${ECR_REGISTRY}/${ECR_FRONTEND_REPO}:${IMAGE_TAG}"
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
