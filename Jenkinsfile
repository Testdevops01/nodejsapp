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
        stage('Create EKS Cluster with AWS CLI') {
            steps {
                script {
                    echo "🚀 Creating EKS cluster: ${EKS_CLUSTER_NAME}"
                    sh """
                        # Check if cluster already exists
                        if aws eks describe-cluster --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION} 2>/dev/null; then
                            echo "✅ Cluster ${EKS_CLUSTER_NAME} already exists"
                        else
                            echo "Creating new EKS cluster using AWS CLI..."
                            
                            # Get default VPC and subnets
                            VPC_ID=\$(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query "Vpcs[0].VpcId" --output text)
                            echo "Using VPC: \$VPC_ID"
                            
                            SUBNET_IDS=\$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=\$VPC_ID" --query "Subnets[0:2].SubnetId" --output text | tr '\\n' ',' | sed 's/,\$//')
                            echo "Using subnets: \$SUBNET_IDS"
                            
                            # Create EKS cluster
                            aws eks create-cluster \\
                                --name ${EKS_CLUSTER_NAME} \\
                                --region ${AWS_REGION} \\
                                --kubernetes-version 1.28 \\
                                --role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/JenkinsWorkerRole \\
                                --resources-vpc-config subnetIds="\$SUBNET_IDS"
                            
                            echo "✅ EKS cluster creation started (takes 10-15 minutes)"
                        fi
                    """
                }
            }
        }

        stage('Wait for EKS Control Plane') {
            steps {
                script {
                    echo "⏳ Waiting for EKS control plane to be ready..."
                    sh """
                        TIMEOUT=1200
                        INTERVAL=30
                        ELAPSED=0
                        
                        while [ \$ELAPSED -lt \$TIMEOUT ]; do
                            STATUS=\$(aws eks describe-cluster --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION} --query "cluster.status" --output text 2>/dev/null || echo "CREATING")
                            
                            if [ "\$STATUS" = "ACTIVE" ]; then
                                echo "🎉 EKS control plane is ACTIVE!"
                                break
                            elif [ "\$STATUS" = "FAILED" ]; then
                                echo "❌ Cluster creation failed"
                                exit 1
                            else
                                echo "Control plane status: \$STATUS (\$ELAPSED seconds)"
                                sleep \$INTERVAL
                                ELAPSED=\$((ELAPSED + INTERVAL))
                            fi
                        done
                        
                        if [ "\$STATUS" != "ACTIVE" ]; then
                            echo "❌ Timeout waiting for control plane"
                            exit 1
                        fi
                    """
                }
            }
        }

        stage('Create EKS Node Group') {
            steps {
                script {
                    echo "🖥️ Creating EKS Node Group..."
                    sh """
                        # Get subnets again
                        VPC_ID=\$(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query "Vpcs[0].VpcId" --output text)
                        SUBNET_IDS=\$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=\$VPC_ID" --query "Subnets[0:2].SubnetId" --output text | tr '\\n' ',' | sed 's/,\$//')
                        
                        # Create node group
                        aws eks create-nodegroup \\
                            --cluster-name ${EKS_CLUSTER_NAME} \\
                            --nodegroup-name workers \\
                            --instance-types t3.medium \\
                            --scaling-config minSize=1,maxSize=3,desiredSize=2 \\
                            --subnets "\$SUBNET_IDS" \\
                            --node-role arn:aws:iam::${AWS_ACCOUNT_ID}:role/JenkinsWorkerRole \\
                            --region ${AWS_REGION}
                        
                        echo "✅ Node group creation started"
                    """
                }
            }
        }

        stage('Wait for Node Group') {
            steps {
                script {
                    echo "⏳ Waiting for node group to be ready..."
                    sh """
                        TIMEOUT=900
                        INTERVAL=30
                        ELAPSED=0
                        
                        while [ \$ELAPSED -lt \$TIMEOUT ]; do
                            STATUS=\$(aws eks describe-nodegroup --cluster-name ${EKS_CLUSTER_NAME} --nodegroup-name workers --region ${AWS_REGION} --query "nodegroup.status" --output text 2>/dev/null || echo "CREATING")
                            
                            if [ "\$STATUS" = "ACTIVE" ]; then
                                echo "🎉 Node group is ACTIVE!"
                                break
                            elif [ "\$STATUS" = "CREATE_FAILED" ]; then
                                echo "❌ Node group creation failed"
                                exit 1
                            else
                                echo "Node group status: \$STATUS (\$ELAPSED seconds)"
                                sleep \$INTERVAL
                                ELAPSED=\$((ELAPSED + INTERVAL))
                            fi
                        done
                        
                        if [ "\$STATUS" != "ACTIVE" ]; then
                            echo "❌ Timeout waiting for node group"
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
                        aws eks update-kubeconfig \\
                            --region ${AWS_REGION} \\
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
                        # Create deployment
                        kubectl create deployment ${APP_NAME} --image=${DOCKER_IMAGE} --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Expose service
                        kubectl expose deployment ${APP_NAME} \\
                            --port=80 \\
                            --target-port=3000 \\
                            --type=LoadBalancer \\
                            --name=${APP_NAME}-service \\
                            --dry-run=client -o yaml | kubectl apply -f -
                        
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
