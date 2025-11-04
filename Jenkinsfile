pipeline {
    agent any

    environment {
        CLUSTER_NAME = "nodejs-eks-cluster"
        AWS_REGION   = "us-east-1"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
                echo "✅ Code checkout completed"
            }
        }

        stage('Test AWS Credentials') {
            steps {
                withCredentials([[ 
                    $class: 'AmazonWebServicesCredentialsBinding', 
                    credentialsId: 'aws-creds'
                ]]) {
                    sh 'aws sts get-caller-identity'
                }
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-login',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                    echo "🐳 Building and pushing Docker image..."
                    docker login -u "$DOCKER_USER" -p "$DOCKER_PASS"
                    docker build -t anusha987/nodejsapp:latest .
                    docker push anusha987/nodejsapp:latest
                    echo "✅ Docker image pushed successfully"
                    '''
                }
            }
        }

        stage('Create EKS Cluster') {
    steps {
        withCredentials([[
            $class: 'AmazonWebServicesCredentialsBinding',
            credentialsId: 'aws-creds'
        ]]) {
            sh '''
            echo "🚀 Checking if EKS cluster already exists..."
            if eksctl get cluster --name nodejs-eks-cluster --region us-east-1 >/dev/null 2>&1; then
                echo "✅ EKS cluster already exists — skipping creation."
            else
                echo "🚀 Creating new EKS cluster..."
                eksctl create cluster \
                    --name nodejs-eks-cluster \
                    --region us-east-1 \
                    --nodegroup-name workers \
                    --node-type t3.medium \
                    --nodes 2 \
                    --managed \
                    --version 1.28
            fi
            '''
        	}
    	    }
	}


        stage('Configure Access') {
            steps {
                withCredentials([[ 
                    $class: 'AmazonWebServicesCredentialsBinding', 
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                    echo "🔧 Updating kubeconfig..."
                    aws eks update-kubeconfig --region us-east-1 --name nodejs-eks-cluster
                    echo "✅ kubeconfig updated"
                    '''
                }
            }
        }

        stage('Deploy App') {
            steps {
                sh '''
                echo "🚀 Deploying application to EKS..."
                kubectl apply -f deployment.yaml
                kubectl apply -f service.yaml
                echo "✅ Application deployed"
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                echo "🔍 Verifying deployment..."
                kubectl get pods
                kubectl get svc
                echo "✅ Verification complete"
                '''
            }
        }

        stage('Get App URL') {
            steps {
                sh '''
                echo "🌐 Fetching LoadBalancer URL..."
                kubectl get svc nodejsapp-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
                '''
            }
        }
    }

    post {
        failure {
            echo "❌ Pipeline failed! Check logs above."
        }
        success {
            echo "🎉 Pipeline completed successfully!"
        }
    }
}

