pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'anusha987'
        APP_NAME = 'nodejsapp'
        EKS_CLUSTER_NAME = 'nodejs-eks-cluster'
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
        
        stage('Create EKS Cluster if Not Exists') {
            options {
                timeout(time: 45, unit: 'MINUTES')  // EKS creation can take 15-30 minutes
            }
            steps {
                withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                    script {
                        // Check if cluster exists
                        def clusterExists = sh(
                            script: """
                                aws eks describe-cluster --name $EKS_CLUSTER_NAME --region $AWS_REGION > /dev/null 2>&1 && echo "exists" || echo "not_exists"
                            """,
                            returnStdout: true
                        ).trim()
                        
                        if (clusterExists == "not_exists") {
                            echo "🚀 Creating EKS cluster: $EKS_CLUSTER_NAME"
                            sh """
                                eksctl create cluster \
                                    --name $EKS_CLUSTER_NAME \
                                    --region $AWS_REGION \
                                    --nodegroup-name workers \
                                    --node-type t3.medium \
                                    --nodes 2 \
                                    --nodes-min 1 \
                                    --nodes-max 3 \
                                    --managed \
                                    --external-dns-access \
                                    --full-ecr-access \
                                    --appmesh-access \
                                    --alb-ingress-access \
                                    --auto-kubeconfig
                            """
                            echo "✅ EKS cluster created successfully"
                        } else {
                            echo "✅ EKS cluster already exists - skipping creation"
                        }
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
                    """
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
                            echo "🌐 LoadBalancer URL:"
                            kubectl get svc nodejs-app -o jsonpath="{.status.loadBalancer.ingress[0].hostname}" || echo "LoadBalancer not ready yet"
                        """
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo '🎉 Pipeline completed successfully!'
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
