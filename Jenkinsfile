pipeline {
    agent any
    
    environment {
        AWS_REGION = 'us-east-1'
        CLUSTER_NAME = 'nodejs-eks-cluster'
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
                sh 'echo "✅ Code checkout completed"'
            }
        }
        
        stage('Create EKS Cluster') {
            steps {
                script {
                    sh """
                    echo "🚀 Creating EKS cluster..."
                    eksctl create cluster \\
                        --name ${CLUSTER_NAME} \\
                        --region ${AWS_REGION} \\
                        --nodegroup-name workers \\
                        --node-type t3.medium \\
                        --nodes 2 \\
                        --managed \\
                        --version 1.28
                    echo "✅ EKS cluster created"
                    """
                }
            }
        }
        
        stage('Configure Access') {
            steps {
                script {
                    sh """
                    echo "🔧 Configuring kubectl access..."
                    aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}
                    sleep 60
                    kubectl get nodes
                    """
                }
            }
        }
        
        stage('Deploy App') {
            steps {
                script {
                    sh """
                    echo "📦 Deploying application..."
                    kubectl apply -f deployment.yaml
                    sleep 30
                    kubectl get pods
                    """
                }
            }
        }
        
        stage('Verify') {
            steps {
                script {
                    sh """
                    echo "🔍 Verifying deployment..."
                    kubectl rollout status deployment/nodejs-app --timeout=300s
                    echo "✅ Deployment successful"
                    """
                }
            }
        }
        
        stage('Get URL') {
            steps {
                script {
                    sh """
                    echo "🌐 Getting application URL..."
                    sleep 30
                    kubectl get service nodejs-app
                    echo "📋 Run this command to get your URL:"
                    echo "kubectl get service nodejs-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'"
                    """
                }
            }
        }
    }
    
    post {
        success {
            echo "🎉 Pipeline completed successfully!"
            echo "Your EKS cluster is ready and application is deployed!"
        }
    }
}
