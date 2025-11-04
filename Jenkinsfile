pipeline {
    agent any
    
    environment {
        AWS_ACCOUNT_ID = '843559766730'
        AWS_REGION = 'us-east-1'
        ECR_REPO = 'nodejs-eks-app'
        CLUSTER_NAME = 'nodejs-eks-cluster'
        DOCKER_IMAGE = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:latest"
    }
    
    stages {
        stage('Checkout Source') {
            steps {
                checkout scm
                sh 'echo "Code checked out successfully"'
                sh 'ls -la'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image..."
                    sh """
                    docker build -t ${ECR_REPO}:latest .
                    """
                }
            }
        }
        
        stage('Login to AWS ECR') {
            steps {
                script {
                    sh """
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    """
                }
            }
        }
        
        stage('Create ECR Repository') {
            steps {
                script {
                    try {
                        sh """
                        aws ecr describe-repositories --repository-names ${ECR_REPO} --region ${AWS_REGION}
                        echo "ECR repository already exists"
                        """
                    } catch (Exception e) {
                        echo "Creating ECR repository: ${ECR_REPO}"
                        sh """
                        aws ecr create-repository --repository-name ${ECR_REPO} --region ${AWS_REGION}
                        """
                    }
                }
            }
        }
        
        stage('Push to ECR') {
            steps {
                script {
                    sh """
                    docker tag ${ECR_REPO}:latest ${DOCKER_IMAGE}
                    docker push ${DOCKER_IMAGE}
                    echo "Image pushed: ${DOCKER_IMAGE}"
                    """
                }
            }
        }
        
        stage('Create EKS Cluster') {
            steps {
                script {
                    // Check if cluster exists
                    def clusterExists = sh(
                        script: "aws eks list-clusters --region ${AWS_REGION} --query 'clusters' --output text | grep ${CLUSTER_NAME} || true",
                        returnStatus: true
                    )
                    
                    if (clusterExists != 0) {
                        echo "Creating EKS cluster: ${CLUSTER_NAME}"
                        sh """
                        eksctl create cluster \
                        --name ${CLUSTER_NAME} \
                        --region ${AWS_REGION} \
                        --nodegroup-name workers \
                        --node-type t3.medium \
                        --nodes 2 \
                        --nodes-min 1 \
                        --nodes-max 3 \
                        --managed
                        """
                    } else {
                        echo "EKS cluster ${CLUSTER_NAME} already exists"
                    }
                }
            }
        }
        
        stage('Configure Kubectl') {
            steps {
                script {
                    sh """
                    aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}
                    kubectl cluster-info
                    """
                    
                    // Wait for nodes to be ready
                    retry(5) {
                        sh """
                        kubectl get nodes
                        kubectl wait --for=condition=Ready nodes --all --timeout=60s
                        """
                        sleep 10
                    }
                }
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                script {
                    // Update deployment with actual ECR image
                    sh """
                    sed -i 's|kushakumar/nodejsapp-9.0:latest|${DOCKER_IMAGE}|g' deployment.yaml
                    """
                    
                    echo "Deploying application to EKS..."
                    sh """
                    kubectl apply -f deployment.yaml
                    """
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    echo "Waiting for deployment to complete..."
                    retry(10) {
                        sh """
                        kubectl rollout status deployment/nodejs-app --timeout=180s
                        """
                    }
                    
                    echo "Checking deployment status..."
                    sh """
                    kubectl get deployments,svc,pods -o wide
                    """
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    echo "Testing application..."
                    retry(12) {
                        sh """
                        LB_URL=\$(kubectl get service nodejs-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
                        if [ -z "\$LB_URL" ]; then
                            echo "LoadBalancer not ready yet..."
                            exit 1
                        else
                            echo "Application URL: http://\$LB_URL"
                            curl -f http://\$LB_URL/ || exit 1
                            echo "✅ Application is working!"
                        fi
                        """
                        sleep 15
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo "Pipeline execution completed - ${currentBuild.currentResult}"
        }
        success {
            echo "✅ Deployment Successful!"
            script {
                def lbUrl = sh(
                    script: "kubectl get service nodejs-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'",
                    returnStdout: true
                ).trim()
                echo "🎉 Your app is live at: http://${lbUrl}"
            }
        }
        failure {
            echo "❌ Deployment Failed!"
        }
    }
}
