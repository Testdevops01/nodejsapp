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


        stage('Create EKS Cluster') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh """
                    echo "🚀 Creating EKS cluster..."
                    eksctl create cluster \
                        --name ${CLUSTER_NAME} \
                        --region ${AWS_REGION} \
                        --nodegroup-name workers \
                        --node-type t3.medium \
                        --nodes 2 \
                        --managed \
                        --version 1.28
                    echo "✅ EKS cluster created successfully"
                    """
                }
            }
        }

        stage('Configure Kubernetes Access') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh """
                    echo "🔧 Configuring kubectl access..."
                    aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}

                    # Quick check - if nodes exist, cluster is ready
                    if kubectl get nodes --no-headers 2>/dev/null; then
                        echo "✅ Cluster is ready and accessible"
                    else
                        echo "⏳ Waiting for cluster to be ready..."
                        sleep 30
                        kubectl get nodes
                    fi
                    """
                }
            }
        }

        stage('Deploy/Redeploy Application') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh """
                    echo "📦 Deploying application..."

                    # Delete existing deployment if exists (for clean redeploy)
                    kubectl delete -f deployment.yaml --ignore-not-found=true
                    sleep 5

                    # Apply new deployment
                    kubectl apply -f deployment.yaml
                    echo "✅ Application deployment initiated"
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh """
                echo "🔍 Verifying deployment..."
                kubectl get pods
                kubectl get svc
                """
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed — check the logs above for details."
        }
    }
}

