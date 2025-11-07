pipeline {
    agent any
    
    environment {
        // AWS Configuration
        AWS_ACCOUNT_ID = '843559766730'
        AWS_REGION = 'us-east-1'
        ECR_REPO_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/my-app"
        EKS_CLUSTER_NAME = 'eks-demo'  // Your actual cluster name
        
        // Application Configuration
        APP_NAME = 'my-app'
        K8S_NAMESPACE = 'default'
        DOCKER_IMAGE = "${ECR_REPO_URI}:${BUILD_NUMBER}"
        DEPLOY_ENVIRONMENT = 'dev'  // Hardcoded to dev for learning
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image for ${DEPLOY_ENVIRONMENT}: ${DOCKER_IMAGE}"
                    sh """
                        docker build -t ${DOCKER_IMAGE} .
                        docker tag ${DOCKER_IMAGE} ${ECR_REPO_URI}:latest
                    """
                }
            }
        }
        
        stage('Push to ECR') {
            steps {
                script {
                    echo "Pushing to ECR using IAM Role..."
                    sh """
                        # Login to ECR using IAM Role
                        aws ecr get-login-password --region ${AWS_REGION} | \\
                        docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                        
                        # Push images
                        docker push ${DOCKER_IMAGE}
                        docker push ${ECR_REPO_URI}:latest
                        
                        echo "✅ Images pushed successfully!"
                    """
                }
            }
        }
        
        stage('Verify ECR Push') {
            steps {
                script {
                    echo "Verifying ECR push..."
                    sh """
                        aws ecr list-images \\
                            --repository-name my-app \\
                            --region ${AWS_REGION} \\
                            --query 'imageIds[].imageTag' \\
                            --output table
                    """
                }
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                script {
                    echo "Deploying to EKS: ${EKS_CLUSTER_NAME}"
                    sh """
                        # Configure kubectl for EKS
                        aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}
                        
                        # Create namespace if it doesn't exist
                        kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Update or create deployment
                        kubectl set image deployment/${APP_NAME} ${APP_NAME}=${DOCKER_IMAGE} -n ${K8S_NAMESPACE} --record || \\
                        kubectl create deployment ${APP_NAME} --image=${DOCKER_IMAGE} -n ${K8S_NAMESPACE}
                        
                        # Wait for rollout
                        kubectl rollout status deployment/${APP_NAME} -n ${K8S_NAMESPACE} --timeout=300s
                        
                        echo "✅ Deployment completed to ${DEPLOY_ENVIRONMENT}!"
                    """
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    echo "Verifying deployment..."
                    sh """
                        echo "=== Deployment Status ==="
                        kubectl get deployments -n ${K8S_NAMESPACE}
                        
                        echo ""
                        echo "=== Pods Status ==="
                        kubectl get pods -n ${K8S_NAMESPACE} | grep ${APP_NAME} || echo "No pods found yet"
                        
                        echo ""
                        echo "=== Services ==="
                        kubectl get services -n ${K8S_NAMESPACE} | grep ${APP_NAME} || echo "No services found"
                    """
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "Pipeline execution completed: ${currentBuild.currentResult}"
                // Clean up Docker images
                sh """
                    docker rmi ${DOCKER_IMAGE} || true
                    docker rmi ${ECR_REPO_URI}:latest || true
                """
            }
        }
        success {
            echo "🎉 SUCCESS! Pipeline completed using IAM Role - No AWS credentials stored!"
            echo "🚀 Application deployed to ${DEPLOY_ENVIRONMENT} environment"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}
