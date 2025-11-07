pipeline {
    agent any
    
    environment {
        AWS_ACCOUNT_ID = '843559766730'
        AWS_REGION = 'us-east-1'
        EKS_CLUSTER_NAME = 'jenkins-eks-demo'
    }
    
    stages {
        stage('Create EKS Cluster') {
            steps {
                script {
                    echo "Creating EKS cluster..."
                    sh '''
                        # Create cluster with default VPC and subnets
                        aws eks create-cluster \
                            --name jenkins-eks-demo \
                            --version 1.30 \
                            --region us-east-1 \
                            --resources-vpc-config subnetIds=subnet-0774ed98cadbf8fba
                        
                        echo "Cluster creation started - wait 15 minutes"
                    '''
                }
            }
        }
        
        stage('Wait for Cluster') {
            steps {
                script {
                    echo "Waiting for cluster..."
                    sh '''
                        # Simple wait loop
                        for i in {1..30}; do
                            STATUS=$(aws eks describe-cluster --name jenkins-eks-demo --region us-east-1 --query "cluster.status" --output text 2>/dev/null || echo "CREATING")
                            echo "Status: $STATUS"
                            if [ "$STATUS" = "ACTIVE" ]; then
                                echo "Cluster ready!"
                                break
                            fi
                            sleep 30
                        done
                    '''
                }
            }
        }
    }
}
        
        stage('Configure kubectl') {
            steps {
                script {
                    echo "🔧 Configuring kubectl access..."
                    sh """
                        # Update kubeconfig
                        aws eks update-kubeconfig \\
                            --region ${AWS_REGION} \\
                            --name ${EKS_CLUSTER_NAME}
                        
                        # Test cluster access
                        echo "=== Testing cluster access ==="
                        kubectl cluster-info
                        
                        echo ""
                        echo "=== Cluster nodes ==="
                        kubectl get nodes || echo "No worker nodes yet - may need to add node group"
                    """
                }
            }
        }
        
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
                    echo "🐳 Building Docker image..."
                    sh """
                        docker build -t ${DOCKER_IMAGE} .
                        docker tag ${DOCKER_IMAGE} ${ECR_REPO_URI}:latest
                        echo "✅ Image built: ${DOCKER_IMAGE}"
                    """
                }
            }
        }
        
        stage('Push to ECR') {
            steps {
                script {
                    echo "📤 Pushing to ECR..."
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | \\
                        docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                        
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
                    echo "🔍 Verifying ECR push..."
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
                    echo "🚀 Deploying application to EKS..."
                    sh """
                        # Create namespace
                        kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Create deployment
                        kubectl create deployment ${APP_NAME} \\
                            --image=${DOCKER_IMAGE} \\
                            -n ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Expose as service
                        kubectl expose deployment ${APP_NAME} \\
                            --port=80 \\
                            --target-port=3000 \\
                            --type=LoadBalancer \\
                            --name=${APP_NAME}-service \\
                            -n ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Wait for rollout
                        kubectl rollout status deployment/${APP_NAME} -n ${K8S_NAMESPACE} --timeout=300s
                        
                        echo "✅ Application deployed!"
                    """
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    echo "🔍 Verifying deployment..."
                    sh """
                        echo "=== Deployment Status ==="
                        kubectl get deployments -n ${K8S_NAMESPACE} -o wide
                        
                        echo ""
                        echo "=== Pods Status ==="
                        kubectl get pods -n ${K8S_NAMESPACE} -o wide
                        
                        echo ""
                        echo "=== Services ==="
                        kubectl get services -n ${K8S_NAMESPACE} -o wide
                        
                        echo ""
                        echo "=== LoadBalancer URL ==="
                        kubectl get service ${APP_NAME}-service -n ${K8S_NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || echo "LoadBalancer provisioning..."
                    """
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "🏁 Pipeline ${currentBuild.currentResult}"
                // Cleanup
                sh """
                    docker rmi ${DOCKER_IMAGE} || true
                    docker rmi ${ECR_REPO_URI}:latest || true
                """
            }
        }
        success {
            echo """
            🎉 COMPLETE SUCCESS! 🎉
            
            ✅ EKS Cluster: Created and ready
            ✅ ECR: Images pushed using IAM Role  
            ✅ Application: Deployed to Kubernetes
            ✅ Security: No AWS credentials stored
            
            Note: The EKS cluster has no worker nodes yet.
            To add nodes, create a node group in AWS Console or use:
            aws eks create-nodegroup --cluster-name ${EKS_CLUSTER_NAME} ...
            """
        }
        failure {
            echo "❌ Pipeline failed - check logs for details"
        }
    }
}
