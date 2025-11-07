pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'anusha987'
        APP_NAME = 'nodejsapp'
        EKS_CLUSTER_NAME = 'nodejs-eks-cluster-dev'
        AWS_REGION = 'us-east-1'
        K8S_NAMESPACE = 'default'
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
                echo '✅ Code checkout completed'
            }
        }
        
        stage('Test AWS Credentials') {
            steps {
                withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                    sh '''
                        echo "Testing AWS credentials..."
                        aws sts get-caller-identity
                    '''
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "🐳 Logging into Docker Hub..."
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        
                        echo "🐳 Building Docker image..."
                        docker build -t $DOCKER_REGISTRY/$APP_NAME:latest .
                        
                        echo "✅ Docker image built successfully"
                    '''
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "📤 Pushing Docker image..."
                        docker push $DOCKER_REGISTRY/$APP_NAME:latest
                        echo "✅ Docker image pushed successfully"
                    '''
                }
            }
        }
        
        stage('Force Cleanup Existing Stacks') {
            steps {
                withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                    script {
                        echo "🧹 Force cleaning up any existing CloudFormation stacks..."
                        sh """
                            # Delete any existing stacks
                            aws cloudformation delete-stack --stack-name eksctl-nodejs-eks-cluster-nodegroup-workers --region $AWS_REGION 2>/dev/null || true
                            aws cloudformation delete-stack --stack-name eksctl-nodejs-eks-cluster-cluster --region $AWS_REGION 2>/dev/null || true
                            
                            # Wait for deletions
                            sleep 30
                            
                            # Force delete using eksctl if needed
                            eksctl delete cluster --region $AWS_REGION --name $EKS_CLUSTER_NAME --force --wait 2>/dev/null || true
                            
                            echo "✅ Cleanup completed"
                        """
                    }
                }
            }
        }
        
        stage('Wait for Cleanup') {
            steps {
                script {
                    echo "⏳ Waiting for cleanup to complete..."
                    sleep 60
                }
            }
        }
        
        stage('Create EKS Cluster') {
            options {
                timeout(time: 45, unit: 'MINUTES')
            }
            steps {
                withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                    script {
                        echo "🚀 Creating EKS cluster: $EKS_CLUSTER_NAME"
                        sh """
                            # Create EKS cluster with a different approach
                            eksctl create cluster \
                                --name $EKS_CLUSTER_NAME \
                                --region $AWS_REGION \
                                --version 1.28 \
                                --nodegroup-name standard-workers \
                                --node-type t3.medium \
                                --nodes 2 \
                                --nodes-min 1 \
                                --nodes-max 3 \
                                --managed \
                                --asg-access \
                                --full-ecr-access \
                                --auto-kubeconfig \
                                --verbose 4
                        """
                        echo "✅ EKS cluster created successfully"
                    }
                }
            }
        }
        
        stage('Configure EKS Access') {
            steps {
                withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                    sh """
                        echo "🔧 Updating kubeconfig..."
                        aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME
                        echo "✅ kubeconfig updated"
                        
                        # Verify cluster access
                        echo "🔍 Testing cluster access..."
                        kubectl cluster-info
                        echo "📊 Checking nodes..."
                        kubectl get nodes
                    """
                }
            }
        }
        
        stage('Setup Docker Registry Secret') {
            steps {
                withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                    script {
                        withCredentials([usernamePassword(
                            credentialsId: 'docker-creds',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )]) {
                            sh """
                                aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME
                                
                                # Create docker registry secret
                                kubectl create secret docker-registry docker-credentials \
                                    --docker-server=https://index.docker.io/v1/ \
                                    --docker-username=$DOCKER_USER \
                                    --docker-password=$DOCKER_PASS \
                                    --docker-email=test@example.com \
                                    --dry-run=client -o yaml | kubectl apply -f -
                                
                                echo "✅ Docker registry secret created"
                            """
                        }
                    }
                }
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                    sh """
                        echo "🚀 Deploying application to EKS..."
                        aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME
                        kubectl apply -f deployment.yaml
                        echo "✅ Deployment completed"
                    """
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                    sh """
                        echo "🔍 Checking deployment status..."
                        aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME
                        
                        echo "⏳ Waiting for deployment to complete..."
                        kubectl rollout status deployment/nodejs-app -n $K8S_NAMESPACE --timeout=300s
                        
                        echo "📊 Checking pods..."
                        kubectl get pods -n $K8S_NAMESPACE -l app=nodejs-app
                        
                        echo "🌐 Checking services..."
                        kubectl get svc -n $K8S_NAMESPACE
                        
                        echo "✅ Verification complete"
                    """
                }
            }
        }
        
        stage('Get Application URL') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                        sh """
                            aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME
                            echo "🌐 Application URL:"
                            kubectl get svc nodejs-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' || echo "LoadBalancer not ready yet"
                        """
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo '🎉 Pipeline completed successfully!'
            script {
                echo "🚀 Your complete CI/CD pipeline has built everything from scratch!"
                echo "✅ Docker image built and pushed"
                echo "✅ EKS cluster created"
                echo "✅ Nodegroup created" 
                echo "✅ Application deployed to Kubernetes"
            }
        }
        failure {
            echo '❌ Pipeline failed!'
        }
        always {
            echo '🧹 Cleaning up...'
            sh 'docker system prune -f || true'
        }
    }
}
