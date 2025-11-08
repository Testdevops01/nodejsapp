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
                        # Check if cluster already exists
                        if aws eks describe-cluster --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION} 2>/dev/null; then
                            echo "✅ Cluster ${EKS_CLUSTER_NAME} already exists"
                        else
                            echo "Creating new EKS cluster..."
                            eksctl create cluster \
                                --name ${EKS_CLUSTER_NAME} \
                                --region ${AWS_REGION} \
                                --version 1.30 \
                                --nodegroup-name workers \
                                --node-type t3.medium \
                                --nodes 2 \
                                --nodes-min 1 \
                                --nodes-max 3 \
                                --managed \
                                --auto-kubeconfig
                            
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
                        docker tag ${DOCKER_IMAGE} ${ECR_REPO_URI}:latest
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
                        
                        # Push both tags
                        docker push ${DOCKER_IMAGE}
                        docker push ${ECR_REPO_URI}:latest
                        
                        echo "✅ Images pushed to ECR"
                        
                        # Verify push
                        echo "=== ECR Images ==="
                        aws ecr list-images --repository-name my-app --region ${AWS_REGION} --query 'imageIds[*].imageTag' --output table
                    """
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    echo "🚀 Deploying to EKS..."
                    sh """
                        # Create updated deployment manifest with ECR image
                        cat > k8s-deployment-${BUILD_NUMBER}.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nodejs-app
  template:
    metadata:
      labels:
        app: nodejs-app
    spec:
      containers:
      - name: nodejs-app
        image: "${DOCKER_IMAGE}"
        ports:
          - containerPort: 3000
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: nodejs-app
  namespace: default
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  selector:
    app: nodejs-app
  type: LoadBalancer
  ports:
  - name: http
    targetPort: 3000
    port: 80
EOF

                        # Apply the deployment
                        kubectl apply -f k8s-deployment-${BUILD_NUMBER}.yaml
                        
                        # Wait for rollout to complete
                        kubectl rollout status deployment/nodejs-app --timeout=300s
                        
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
                        kubectl get deployment nodejs-app

                        echo ""
                        echo "=== Pods Status ==="
                        kubectl get pods -l app=nodejs-app

                        echo ""
                        echo "=== Services ==="
                        kubectl get service nodejs-app

                        echo ""
                        echo "=== Application Logs ==="
                        kubectl logs -l app=nodejs-app --tail=10 || echo "Logs not available yet"

                        echo ""
                        echo "=== LoadBalancer URL ==="
                        SERVICE_URL=\$(kubectl get service nodejs-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null)
                        if [ ! -z "\$SERVICE_URL" ]; then
                            echo "🌐 Your Node.js app is available at: http://\$SERVICE_URL"
                            echo ""
                            echo "⏳ Testing application (waiting for LoadBalancer to be ready)..."
                            sleep 30
                            echo "Testing HTTP response..."
                            curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://\$SERVICE_URL || echo "Application still starting up"
                        else
                            echo "LoadBalancer provisioning... Check in a few minutes with:"
                            echo "kubectl get service nodejs-app"
                        fi
                    """
                }
            }
        }
    }

    post {
        always {
            echo "🏁 Pipeline ${currentBuild.currentResult}"
            sh """
                # Cleanup Docker images
                docker rmi ${DOCKER_IMAGE} || true
                docker rmi ${ECR_REPO_URI}:latest || true
                docker system prune -f || true
                
                # Cleanup temporary files
                rm -f k8s-deployment-*.yaml || true
            """
        }
        success {
            echo """
            🎉 PIPELINE SUCCESS! 🎉
            
            Your Node.js Express app is now running on EKS!
            
            Quick Commands:
            🔍 Check status: kubectl get all -l app=nodejs-app
            📝 View logs: kubectl logs -l app=nodejs-app -f
            🌐 Get URL: kubectl get service nodejs-app
            📊 Monitor: kubectl top pods -l app=nodejs-app
            
            The EKS cluster will remain running for future deployments.
            """
        }
        failure {
            echo "❌ Pipeline failed. Check the logs above for details."
        }
    }
}
