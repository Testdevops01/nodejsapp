
pipeline {
    agent any
    environment {
        AWS_REGION = 'us-east-1'
        EKS_CLUSTER_NAME = 'nodejs-app-cluster'
        DOCKER_IMAGE = 'kushakumar/nodejsapp-9.0:latest'
        GIT_REPO = 'your-repo-url-here'
    }
    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', 
                    url: "${env.GIT_REPO}",
                    credentialsId: 'your-git-credentials'
            }
        }
        
        stage('Setup AWS CLI and kubectl') {
            steps {
                sh '''
                    # Install kubectl
                    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                    chmod +x ./kubectl
                    sudo mv ./kubectl /usr/local/bin/kubectl
                    
                    # Install eksctl
                    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
                    sudo mv /tmp/eksctl /usr/local/bin
                    
                    # Verify installations
                    eksctl version
                    kubectl version --client
                '''
            }
        }
        
        stage('Create EKS Cluster') {
            steps {
                sh """
                    eksctl create cluster \\
                    --name ${env.EKS_CLUSTER_NAME} \\
                    --region ${env.AWS_REGION} \\
                    --nodegroup-name workers \\
                    --node-type t3.medium \\
                    --nodes 2 \\
                    --nodes-min 1 \\
                    --nodes-max 3 \\
                    --managed \\
                    --ssh-access \\
                    --ssh-public-key your-key-pair \\
                    --version 1.28
                """
            }
        }
        
        stage('Configure kubectl') {
            steps {
                sh """
                    aws eks update-kubeconfig --region ${env.AWS_REGION} --name ${env.EKS_CLUSTER_NAME}
                    kubectl get nodes
                """
            }
        }
        
        stage('Build and Push Docker Image') {
            steps {
                script {
                    // Build Docker image
                    docker.build("${env.DOCKER_IMAGE}")
                    
                    // Push to Docker Hub
                    docker.withRegistry('', 'docker-hub-credentials') {
                        docker.image("${env.DOCKER_IMAGE}").push()
                    }
                }
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                sh """
                    # Apply the single YAML file containing both Deployment and Service
                    kubectl apply -f deployment.yaml
                    
                    # Verify deployment
                    echo "=== Deployments ==="
                    kubectl get deployments
                    
                    echo "=== Services ==="
                    kubectl get services
                    
                    echo "=== Pods ==="
                    kubectl get pods
                """
            }
        }
        
        stage('Test Application') {
            steps {
                sh """
                    # Wait for service to get external IP
                    echo "Waiting for LoadBalancer to be ready..."
                    for i in {1..30}; do
                        if kubectl get service nodejs-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null; then
                            break
                        fi
                        sleep 10
                    done
                    
                    SERVICE_URL=\$(kubectl get service nodejs-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
                    echo "Service URL: http://\${SERVICE_URL}"
                    
                    # Test the application
                    echo "Testing the application..."
                    curl -f http://\${SERVICE_URL} || exit 1
                """
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline execution completed'
        }
        success {
            echo 'Application deployed successfully!'
            sh """
                SERVICE_URL=\$(kubectl get service nodejs-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || echo "not-available")
                echo "Your application is available at: http://\${SERVICE_URL}"
            """
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
