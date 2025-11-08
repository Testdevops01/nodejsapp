pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = '843559766730'
        AWS_REGION = 'us-east-1'
        ECR_REPO_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/my-app"
        EKS_CLUSTER_NAME = 'jenkins-eks-demo'
        APP_NAME = 'nodejs-app'
        DOCKER_IMAGE = "${ECR_REPO_URI}:${BUILD_NUMBER}"
    }

    stages {
        stage('Create EKS Cluster') {
            steps {
                script {
                    echo "🚀 Creating EKS cluster: ${EKS_CLUSTER_NAME}"
                    sh """
                        # Check if cluster exists using basic describe (should work)
                        if aws eks describe-cluster --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION} 2>/dev/null; then
                            echo "✅ Cluster ${EKS_CLUSTER_NAME} already exists"
                        else
                            echo "Creating new EKS cluster..."
                            # Use eksctl without specifying version (it will use default)
                            eksctl create cluster \
                                --name ${EKS_CLUSTER_NAME} \
                                --region ${AWS_REGION} \
                                --nodegroup-name workers \
                                --node-type t3.medium \
                                --nodes 2 \
                                --nodes-min 1 \
                                --nodes-max 3 \
                                --managed
                            
                            echo "✅ EKS cluster creation started (takes 10-15 minutes)"
                        fi
                    """
                }
            }
        }

        stage('Wait for Cluster Ready') {
            steps {
                script {
                    echo "⏳ Waiting for EKS cluster to be ready..."
                    sh """
                        TIMEOUT=1200
                        INTERVAL=30
                        ELAPSED=0
                        
                        while [ \$ELAPSED -lt \$TIMEOUT ]; do
                            STATUS=\$(aws eks describe-cluster --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION} --query "cluster.status" --output text 2>/dev/null || echo "CREATING")
                            
                            if [ "\$STATUS" = "ACTIVE" ]; then
                                echo "🎉 EKS cluster is ACTIVE!"
                                break
                            elif [ "\$STATUS" = "FAILED" ]; then
                                echo "❌ Cluster creation failed"
                                exit 1
                            else
                                echo "Cluster status: \$STATUS (\$ELAPSED seconds)"
                                sleep \$INTERVAL
                                ELAPSED=\$((ELAPSED + INTERVAL))
                            fi
                        done
                        
                        if [ "\$STATUS" != "ACTIVE" ]; then
                            echo "❌ Timeout waiting for cluster"
                            exit 1
                        fi
                    """
                }
            }
        }

        stage('Configure Kubernetes Access') {
            steps {
                script {
                    echo "🔧 Configuring kubectl access..."
                    sh """
                        aws eks update-kubeconfig \
                            --region ${AWS_REGION} \
                            --name ${EKS_CLUSTER_NAME}
                        
                        # Verify cluster access
                        echo "=== Cluster Nodes ==="
                        kubectl get nodes
                        
                        echo "✅ Kubernetes access configured"
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
                        echo "✅ Docker image built: ${DOCKER_IMAGE}"
                    """
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    echo "📤 Pushing to ECR..."
                    sh """
                        # Login to ECR
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO_URI}
                        
                        # Push image
                        docker push ${DOCKER_IMAGE}
                        
                        echo "✅ Image pushed to ECR"
                    """
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    echo "🚀 Deploying to EKS..."
                    sh """
                        # Create deployment using kubectl (simpler approach)
                        kubectl create deployment ${APP_NAME} --image=${DOCKER_IMAGE} --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Expose service
                        kubectl expose deployment ${APP_NAME} \
                            --port=80 \
                            --target-port=3000 \
                            --type=LoadBalancer \
                            --name=${APP_NAME}-service \
                            --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Scale deployment
                        kubectl scale deployment/${APP_NAME} --replicas=2
                        
                        # Wait for rollout
                        kubectl rollout status deployment/${APP_NAME} --timeout=300s
                        
                        echo "✅ Application deployed successfully!"
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
                        kubectl get deployment ${APP_NAME}

                        echo ""
                        echo "=== Pods Status ==="
                        kubectl get pods -l app=${APP_NAME}

                        echo ""
                        echo "=== Services ==="
                        kubectl get service ${APP_NAME}-service

                        echo ""
                        echo "=== LoadBalancer URL ==="
                        SERVICE_URL=\$(kubectl get service ${APP_NAME}-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null)
                        if [ ! -z "\$SERVICE_URL" ]; then
                            echo "🌐 Your Node.js app is available at: http://\$SERVICE_URL"
                            echo "⏳ Waiting for LoadBalancer to be ready..."
                            sleep 30
                            echo "Testing application..."
                            curl -s http://\$SERVICE_URL | head -5 || echo "Application still starting up"
                        else
                            echo "LoadBalancer provisioning..."
                        fi
                    """
                }
            }
        }
    }

    post {
        always {
            echo "🏁 Pipeline ${currentBuild.currentResult}"
        }
        success {
            echo "🎉 Pipeline SUCCESS! Your Node.js app is running on EKS."
        }
        failure {
            echo "❌ Pipeline failed. Check logs above for details."
        }
    }
}
