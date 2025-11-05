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
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            echo '🐳 Logging into Docker Hub...'
                            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                            
                            echo '🐳 Building Docker image...'
                            docker build -t \$DOCKER_REGISTRY/\$APP_NAME:latest -t \$DOCKER_REGISTRY/\$APP_NAME:\${GIT_COMMIT:0:8} .
                            
                            echo '✅ Docker image built successfully'
                        """
                    }
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            echo '📤 Pushing Docker image...'
                            docker push \$DOCKER_REGISTRY/\$APP_NAME:latest
                            docker push \$DOCKER_REGISTRY/\$APP_NAME:\${GIT_COMMIT:0:8}
                            echo '✅ Docker image pushed successfully'
                        """
                    }
                }
            }
        }
       

	stage('Create EKS Cluster if Not Exists') {
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
                                    --ssh-access \
                                    --ssh-public-key jenkins-key \
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
                        echo '🔧 Updating kubeconfig...'
                        aws eks update-kubeconfig --region \$AWS_REGION --name \$EKS_CLUSTER_NAME
                        echo '✅ kubeconfig updated'
                    """
                }
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                        sh """
                            echo '🚀 Deploying application to EKS...'
                            
                            # Update kubeconfig first
                            aws eks update-kubeconfig --region \$AWS_REGION --name \$EKS_CLUSTER_NAME
                            
                            # Update the deployment with the new image
                            kubectl set image deployment/nodejs-app nodejs-app=\$DOCKER_REGISTRY/\$APP_NAME:latest -n \$K8S_NAMESPACE --record
                            
                            # If deployment doesn't exist, create it
                            if ! kubectl get deployment nodejs-app -n \$K8S_NAMESPACE 2>/dev/null; then
                                echo '📝 Creating new deployment...'
                                kubectl apply -f deployment.yaml
                            fi
                            
                            echo '✅ Deployment completed'
                        """
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                    sh """
                        echo '🔍 Checking deployment status...'
                        
                        # Update kubeconfig
                        aws eks update-kubeconfig --region \$AWS_REGION --name \$EKS_CLUSTER_NAME
                        
                        # Wait for rollout to complete
                        echo '⏳ Waiting for deployment to complete...'
                        kubectl rollout status deployment/nodejs-app -n \$K8S_NAMESPACE --timeout=300s
                        
                        echo '📊 Checking pods...'
                        kubectl get pods -n \$K8S_NAMESPACE -l app=nodejs-app
                        
                        echo '🌐 Checking services...'
                        kubectl get svc -n \$K8S_NAMESPACE
                        
                        echo '✅ Verification complete'
                    """
                }
            }
        }
        
        stage('Get Application URL') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                        // Update kubeconfig first
                        sh """
                            aws eks update-kubeconfig --region \$AWS_REGION --name \$EKS_CLUSTER_NAME
                        """
                        
                        // Wait for LoadBalancer
                        sh """
                            echo '⏳ Waiting for LoadBalancer to be ready...'
                            for i in \$(seq 1 30); do
                                if kubectl get svc nodejs-app -n \$K8S_NAMESPACE -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null | grep -q '.'; then
                                    echo '✅ LoadBalancer is ready'
                                    break
                                fi
                                echo 'Waiting for LoadBalancer...'
                                sleep 10
                            done
                        """
                        
                        // Get the LoadBalancer URL
                        def lbUrl = sh(
                            script: "kubectl get svc nodejs-app -n \$K8S_NAMESPACE -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || echo 'pending'",
                            returnStdout: true
                        ).trim()
                        
                        echo "🌐 Application URL: http://${lbUrl}"
                        
                        // Store the URL as a build artifact
                        sh """
                            echo "Application URL: http://${lbUrl}" > application_url.txt
                            echo "Build: ${env.BUILD_URL}" >> application_url.txt
                        """
                        
                        archiveArtifacts artifacts: 'application_url.txt'
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo '🎉 Pipeline completed successfully!'
            script {
                // Try to get the LB URL for the success message
                def lbUrl = "check-application_url.txt-artifact"
                echo "Application should be available at: http://${lbUrl}"
            }
        }
        failure {
            echo '❌ Pipeline failed!'
        }
        always {
            echo '🧹 Cleaning up...'
            // Clean up Docker images to save space
            sh '''
                docker system prune -f || true
            '''
        }
    }
}
