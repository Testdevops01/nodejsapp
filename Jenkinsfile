pipeline {
    agent any
    
    environment {
        // AWS Configuration
        AWS_ACCOUNT_ID = '843559766730'
        AWS_REGION = 'us-east-1'
        ECR_REPO_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/my-app"
        
        // EKS Cluster Configuration
        EKS_CLUSTER_NAME = 'jenkins-eks-demo'
        EKS_CLUSTER_VERSION = '1.30'
        
        // Application Configuration
        APP_NAME = 'my-app'
        K8S_NAMESPACE = 'default'
        DOCKER_IMAGE = "${ECR_REPO_URI}:${BUILD_NUMBER}"
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 60, unit: 'MINUTES')
    }
    
    stages {
        stage('Create EKS Cluster') {
            steps {
                script {
                    echo "🚀 Creating EKS cluster: ${EKS_CLUSTER_NAME}"
                    sh '''
                        # Get default VPC
                        VPC_ID=$(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query "Vpcs[0].VpcId" --output text)
                        echo "Using default VPC: $VPC_ID"
                        
                        # Get all subnets in the default VPC
                        SUBNET_IDS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[*].SubnetId" --output text | tr '\\n' ',' | sed 's/,$//')
                        echo "Using subnets: $SUBNET_IDS"
                        
                        # Create EKS cluster
                        aws eks create-cluster \
                            --name jenkins-eks-demo \
                            --version 1.30 \
                            --region us-east-1 \
                            --resources-vpc-config subnetIds="$SUBNET_IDS"
                        
                        echo "✅ EKS cluster creation started!"
                        echo "⏳ This will take 10-15 minutes..."
                    '''
                }
            }
        }
        
        stage('Wait for EKS Cluster') {
            steps {
                script {
                    echo "⏳ Waiting for EKS cluster to become active..."
                    sh '''
                        # Wait for cluster with timeout (20 minutes)
                        TIMEOUT=1200
                        INTERVAL=30
                        ELAPSED=0
                        CLUSTER_ACTIVE=false
                        
                        while [ $ELAPSED -lt $TIMEOUT ]; do
                            STATUS=$(aws eks describe-cluster --name jenkins-eks-demo --region us-east-1 --query "cluster.status" --output text 2>/dev/null || echo "CREATING")
                            
                            echo "Cluster status: $STATUS"
                            
                            if [ "$STATUS" = "ACTIVE" ]; then
                                echo "🎉 EKS cluster is now ACTIVE!"
                                CLUSTER_ACTIVE=true
                                break
                            elif [ "$STATUS" = "FAILED" ]; then
                                echo "❌ EKS cluster creation FAILED"
                                echo "Check AWS Console for details"
                                exit 1
                            else
                                echo "Still creating... ($ELAPSED seconds elapsed)"
                                sleep $INTERVAL
                                ELAPSED=$((ELAPSED + INTERVAL))
                            fi
                        done
                        
                        if [ "$CLUSTER_ACTIVE" = "false" ]; then
                            echo "❌ Timeout waiting for EKS cluster after 20 minutes"
                            exit 1
                        fi
                    '''
                }
            }
        }
        
        stage('Configure kubectl') {
            steps {
                script {
                    echo "🔧 Configuring kubectl access..."
                    sh """
                        # Update kubeconfig
                        aws eks update-kubeconfig \
                            --region ${AWS_REGION} \
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
                        aws ecr get-login-password --region ${AWS_REGION} | \
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
                        aws ecr list-images \
                            --repository-name my-app \
                            --region ${AWS_REGION} \
                            --query 'imageIds[].imageTag' \
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
                        kubectl create deployment ${APP_NAME} \
                            --image=${DOCKER_IMAGE} \
                            -n ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Expose as service
                        kubectl expose deployment ${APP_NAME} \
                            --port=80 \
                            --target-port=3000 \
                            --type=LoadBalancer \
                            --name=${APP_NAME}-service \
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
