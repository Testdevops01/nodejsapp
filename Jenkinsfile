pipeline {
    agent any

    environment {
        AWS_CREDENTIALS_ID = 'aws-jenkins-creds'
        REGION = 'us-east-1'
        CLUSTER_NAME = 'jenkins-eks-cluster'
        ECR_REPO = 'my-eks-app'
        IMAGE_TAG = ''
        NAMESPACE = 'default'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Setup Environment') {
            steps {
                script {
                    // Get short commit hash for image tagging
                    shortCommit = sh(
                        returnStdout: true, 
                        script: "git rev-parse --short HEAD"
                    ).trim()
                    env.IMAGE_TAG = "${shortCommit}-${env.BUILD_NUMBER}"
                    
                    // Create unique build ID
                    env.BUILD_ID = sh(
                        returnStdout: true, 
                        script: "echo ${env.BUILD_NUMBER}-$(date +%Y%m%d-%H%M%S)"
                    ).trim()
                }
                echo "Build ID: ${env.BUILD_ID}"
                echo "Image Tag: ${env.IMAGE_TAG}"
            }
        }

        stage('Install Prerequisites') {
            steps {
                script {
                    // Install eksctl if not present
                    sh '''
                        if ! command -v eksctl >/dev/null 2>&1; then
                            echo "Installing eksctl..."
                            curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz"
                            tar -xzf eksctl_$(uname -s)_amd64.tar.gz -C /tmp
                            sudo mv /tmp/eksctl /usr/local/bin/
                            eksctl version
                        else
                            echo "eksctl already installed"
                        fi

                        # Install kubectl if not present
                        if ! command -v kubectl >/dev/null 2>&1; then
                            echo "Installing kubectl..."
                            curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                            sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
                            kubectl version --client
                        else
                            echo "kubectl already installed"
                        fi
                    '''
                }
            }
        }

        stage('Create EKS Cluster') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: "${AWS_CREDENTIALS_ID}",
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        sh """
                            export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                            export AWS_DEFAULT_REGION=${REGION}

                            echo "Checking if EKS cluster ${CLUSTER_NAME} exists..."
                            
                            # Check cluster status with timeout
                            if ! timeout 30s eksctl get cluster --name ${CLUSTER_NAME} --region ${REGION} >/dev/null 2>&1; then
                                echo "🔄 Creating EKS cluster ${CLUSTER_NAME}..."
                                eksctl create cluster \
                                    --name ${CLUSTER_NAME} \
                                    --region ${REGION} \
                                    --nodegroup-name workers \
                                    --node-type t3.medium \
                                    --nodes 2 \
                                    --nodes-min 1 \
                                    --nodes-max 3 \
                                    --managed \
                                    --version 1.28 \
                                    --asg-access \
                                    --full-ecr-access \
                                    --verbose 4
                                
                                echo "✅ Cluster created successfully"
                            else
                                echo "✅ EKS cluster ${CLUSTER_NAME} already exists"
                                
                                # Verify cluster is active
                                CLUSTER_STATUS=$(aws eks describe-cluster --name ${CLUSTER_NAME} --query 'cluster.status' --output text)
                                echo "Cluster status: ${CLUSTER_STATUS}"
                            fi
                        """
                    }
                }
            }
        }

        stage('Build and Push to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: "${AWS_CREDENTIALS_ID}",
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        sh """
                            export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                            export AWS_DEFAULT_REGION=${REGION}

                            # Get account ID
                            ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
                            REPO_URI="\$ACCOUNT_ID.dkr.ecr.${REGION}.amazonaws.com/${ECR_REPO}"
                            
                            echo "Using ECR repository: \$REPO_URI"

                            # Create ECR repo if not exists
                            if ! aws ecr describe-repositories --repository-names ${ECR_REPO} >/dev/null 2>&1; then
                                echo "Creating ECR repository ${ECR_REPO}"
                                aws ecr create-repository --repository-name ${ECR_REPO}
                                sleep 10
                            fi

                            # Login to ECR
                            aws ecr get-login-password --region ${REGION} | \
                                docker login --username AWS --password-stdin \$ACCOUNT_ID.dkr.ecr.${REGION}.amazonaws.com

                            # Build and push image
                            echo "Building Docker image..."
                            docker build -t ${ECR_REPO}:${IMAGE_TAG} .
                            
                            echo "Tagging image..."
                            docker tag ${ECR_REPO}:${IMAGE_TAG} \$REPO_URI:${IMAGE_TAG}
                            docker tag ${ECR_REPO}:${IMAGE_TAG} \$REPO_URI:latest
                            
                            echo "Pushing image to ECR..."
                            docker push \$REPO_URI:${IMAGE_TAG}
                            docker push \$REPO_URI:latest

                            # Save image URI
                            echo "ECR_IMAGE=\$REPO_URI:${IMAGE_TAG}" > ecr.env
                            echo "LATEST_IMAGE=\$REPO_URI:latest" >> ecr.env
                        """
                        
                        // Read image URI back into environment
                        env.ECR_IMAGE = sh(
                            script: "grep ECR_IMAGE ecr.env | cut -d'=' -f2", 
                            returnStdout: true
                        ).trim()
                        echo "✅ Image pushed: ${env.ECR_IMAGE}"
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: "${AWS_CREDENTIALS_ID}",
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        sh """
                            export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                            export AWS_DEFAULT_REGION=${REGION}

                            echo "Configuring kubectl for EKS cluster..."
                            aws eks update-kubeconfig --region ${REGION} --name ${CLUSTER_NAME}

                            # Verify cluster access
                            kubectl cluster-info
                            kubectl get nodes

                            # Create namespace if it doesn't exist
                            if ! kubectl get namespace ${NAMESPACE} >/dev/null 2>&1; then
                                kubectl create namespace ${NAMESPACE}
                            fi

                            echo "Generating Kubernetes manifests..."
                            mkdir -p k8s/generated
                            
                            # Generate deployment.yaml from template
                            sed "s|__IMAGE_PLACEHOLDER__|${ECR_IMAGE}|g" k8s/deployment.yaml.tpl > k8s/generated/deployment.yaml
                            
                            # Apply all manifests
                            echo "Deploying application..."
                            kubectl apply -f k8s/generated/ -n ${NAMESPACE}
                            
                            # Wait for rollout
                            echo "Waiting for deployment to complete..."
                            kubectl rollout status deployment/my-app -n ${NAMESPACE} --timeout=300s
                            
                            # Get deployment info
                            kubectl get deployments,services,pods -n ${NAMESPACE}
                        """
                    }
                }
            }
        }

        stage('Smoke Test') {
            steps {
                script {
                    sh """
                        # Wait a bit for service to be ready
                        sleep 30
                        
                        # Get service URL for testing
                        SERVICE_URL=\$(kubectl get service my-app-service -n ${NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || echo "")
                        
                        if [ -n "\$SERVICE_URL" ]; then
                            echo "Service URL: http://\$SERVICE_URL"
                            echo "Running smoke test..."
                            # Add your smoke test commands here
                            # curl -f http://\$SERVICE_URL/health || exit 1
                        else
                            echo "Service not exposed via LoadBalancer"
                        fi
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline execution completed"
            script {
                // Save deployment information
                sh """
                    echo "Build: ${env.BUILD_ID}" > build-info.txt
                    echo "Image: ${env.ECR_IMAGE}" >> build-info.txt
                    echo "Cluster: ${env.CLUSTER_NAME}" >> build-info.txt
                    echo "Timestamp: $(date)" >> build-info.txt
                """
                archiveArtifacts artifacts: 'build-info.txt', fingerprint: true
            }
        }
        success {
            echo "✅ Pipeline executed successfully!"
            emailext (
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "The build ${env.BUILD_URL} completed successfully.",
                to: "devops-team@company.com"
            )
        }
        failure {
            echo "❌ Pipeline failed!"
            emailext (
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "The build ${env.BUILD_URL} failed. Please check the logs.",
                to: "devops-team@company.com"
            )
        }
    }
}
