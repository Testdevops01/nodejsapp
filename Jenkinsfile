pipeline {
    agent any
    
    environment {
        AWS_ACCOUNT_ID = '843559766730'
        AWS_REGION = 'us-east-1'
        ECR_REPO = 'nodejs-eks-app'
        CLUSTER_NAME = 'nodejs-eks-cluster'
        DOCKER_IMAGE = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
                sh 'echo "✅ Code checkout completed"'
                sh 'ls -la'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                    docker build -t ${ECR_REPO}:${BUILD_NUMBER} .
                    """
                }
            }
        }
        
        stage('Login to ECR') {
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
                    sh """
                    # Create ECR repo if not exists
                    aws ecr describe-repositories --repository-names ${ECR_REPO} --region ${AWS_REGION} || \
                    aws ecr create-repository --repository-name ${ECR_REPO} --region ${AWS_REGION}
                    """
                }
            }
        }
        
        stage('Push to ECR') {
            steps {
                script {
                    sh """
                    docker tag ${ECR_REPO}:${BUILD_NUMBER} ${DOCKER_IMAGE}
                    docker push ${DOCKER_IMAGE}
                    echo "✅ Image pushed: ${DOCKER_IMAGE}"
                    """
                }
            }
        }
        
        stage('Check Cluster Status') {
            steps {
                script {
                    sh """
                    echo "🔍 Checking if EKS cluster exists..."
                    if aws eks describe-cluster --name ${CLUSTER_NAME} --region ${AWS_REGION} > /dev/null 2>&1; then
                        echo "✅ EKS cluster '${CLUSTER_NAME}' already exists"
                        CLUSTER_ACTION="reuse"
                    else
                        echo "🚀 EKS cluster '${CLUSTER_NAME}' not found - creating new cluster"
                        CLUSTER_ACTION="create"
                    fi
                    echo "CLUSTER_ACTION=\${CLUSTER_ACTION}" > cluster_info.txt
                    """
                }
            }
        }
        
        stage('Create EKS Cluster') {
            when {
                expression { 
                    def clusterAction = sh(script: "cat cluster_info.txt | grep CLUSTER_ACTION | cut -d'=' -f2", returnStdout: true).trim()
                    return clusterAction == "create"
                }
            }
            steps {
                script {
                    sh """
                    echo "🚀 Creating EKS cluster (this takes 15-20 minutes)..."
                    eksctl create cluster \\
                        --name ${CLUSTER_NAME} \\
                        --region ${AWS_REGION} \\
                        --nodegroup-name workers \\
                        --node-type t3.medium \\
                        --nodes 2 \\
                        --managed \\
                        --version 1.28
                    
                    echo "✅ EKS cluster created successfully"
                    """
                }
            }
        }
        
        stage('Configure Kubernetes Access') {
            steps {
                script {
                    sh """
                    echo "🔧 Configuring kubectl access..."
                    aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}
                    
                    echo "⏳ Waiting for cluster nodes to be ready..."
                    timeout 300 bash -c 'until kubectl get nodes --no-headers 2>/dev/null | grep -q Ready; do sleep 30; echo "Waiting for nodes..."; done'
                    
                    echo "✅ Cluster Status:"
                    kubectl cluster-info
                    kubectl get nodes -o wide
                    """
                }
            }
        }
        
        stage('Deploy Application') {
            steps {
                script {
                    sh """
                    echo "📦 Deploying application..."
                    
                    # Update deployment with actual ECR image
                    sed -i 's|kushakumar/nodejsapp-9.0:latest|${DOCKER_IMAGE}|g' deployment.yaml
                    
                    # Apply deployment
                    kubectl apply -f deployment.yaml
                    echo "✅ Application deployment initiated"
                    """
                }
            }
        }
        
        stage('Wait for Application') {
            steps {
                script {
                    sh """
                    echo "⏳ Waiting for application to be ready..."
                    kubectl rollout status deployment/nodejs-app --timeout=300s
                    echo "✅ Application deployment completed!"
                    """
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    sh """
                    echo "🔍 Verifying deployment..."
                    kubectl get deployments,svc,pods -o wide
                    
                    # Wait for LoadBalancer
                    echo "⏳ Waiting for LoadBalancer..."
                    for i in {1..30}; do
                        LB_URL=\$(kubectl get service nodejs-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null)
                        if [ -n "\$LB_URL" ]; then
                            echo "✅ LoadBalancer ready: http://\$LB_URL"
                            break
                        else
                            echo "Waiting for LoadBalancer... (\$i/30)"
                            sleep 10
                        fi
                    done
                    """
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    sh """
                    echo "🏥 Performing health check..."
                    LB_URL=\$(kubectl get service nodejs-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
                    if [ -n "\$LB_URL" ]; then
                        echo "🌐 Testing: http://\$LB_URL"
                        if curl -f -m 10 http://\$LB_URL/; then
                            echo "✅ HEALTH CHECK PASSED - Application is working!"
                        else
                            echo "❌ HEALTH CHECK FAILED - Application not responding"
                            exit 1
                        fi
                    else
                        echo "⚠️ LoadBalancer not available yet"
                    fi
                    """
                }
            }
        }
    }
    
    post {
        always {
            sh '''
            echo "=== Cleaning up temporary files ==="
            rm -f cluster_info.txt 2>/dev/null || true
            echo "=== Final Resource Status ==="
            kubectl get all 2>/dev/null || true
            '''
        }
        success {
            script {
                // Read cluster action before file is deleted
                def clusterAction = sh(
                    script: """
                    if [ -f cluster_info.txt ]; then
                        cat cluster_info.txt | grep CLUSTER_ACTION | cut -d'=' -f2
                    else
                        echo "unknown"
                    fi
                    """, 
                    returnStdout: true
                ).trim()
                
                def lbUrl = sh(
                    script: "kubectl get service nodejs-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || echo 'not-available'",
                    returnStdout: true
                ).trim()
                
                if (clusterAction == "reuse") {
                    echo """
                    ✅ FAST DEPLOYMENT COMPLETED!
                    ⏱️  Duration: ~3-5 minutes
                    🌐 Your Application: http://${lbUrl}
                    🐳 Image: ${DOCKER_IMAGE}
                    """
                } else {
                    echo """
                    ✅ FIRST DEPLOYMENT COMPLETED!
                    ⏱️  Duration: ~20-25 minutes  
                    🌐 Your Application: http://${lbUrl}
                    🐳 Image: ${DOCKER_IMAGE}
                    ☸️  Cluster: ${CLUSTER_NAME}
                    """
                }
            }
        }
        failure {
            echo "❌ DEPLOYMENT FAILED"
            script {
                sh """
                echo "=== Debug Information ==="
                echo "1. Checking pods:"
                kubectl get pods -o wide 2>/dev/null || echo "Cannot access cluster"
                
                echo "2. Checking events:"
                kubectl get events --sort-by=.lastTimestamp 2>/dev/null || echo "Cannot get events"
                
                echo "3. Checking service:"
                kubectl describe service nodejs-app 2>/dev/null || echo "Cannot describe service"
                
                echo "4. Checking deployment:"
                kubectl describe deployment nodejs-app 2>/dev/null || echo "Cannot describe deployment"
                """
            }
        }
    }
}
