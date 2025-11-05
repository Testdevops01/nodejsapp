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
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
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
                    withCredentials([[
                        $class: 'UsernamePasswordMultiBinding',
                        credentialsId: 'docker-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    ]]) {
                        sh """
                            echo '🐳 Logging into Docker Hub...'
                            docker login -u $DOCKER_USER -p $DOCKER_PASS
                            
                            echo '🐳 Building Docker image...'
                            docker build -t $DOCKER_REGISTRY/$APP_NAME:latest -t $DOCKER_REGISTRY/$APP_NAME:\${GIT_COMMIT:0:8} .
                            
                            echo '✅ Docker image built successfully'
                        """
                    }
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([[
                        $class: 'UsernamePasswordMultiBinding',
                        credentialsId: 'docker-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    ]]) {
                        sh """
                            echo '📤 Pushing Docker image...'
                            docker push $DOCKER_REGISTRY/$APP_NAME:latest
                            docker push $DOCKER_REGISTRY/$APP_NAME:\${GIT_COMMIT:0:8}
                            echo '✅ Docker image pushed successfully'
                        """
                    }
                }
            }
        }
        
        stage('Configure EKS Access') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh """
                        echo '🔧 Updating kubeconfig...'
                        aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME
                        echo '✅ kubeconfig updated'
                    '''
                }
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                script {
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-creds',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]]) {
                        // Update deployment.yaml with the correct image
                        sh """
                            # Update the deployment with the new image tag
                            kubectl set image deployment/nodejs-app nodejs-app=$DOCKER_REGISTRY/$APP_NAME:latest -n $K8S_NAMESPACE
                            
                            # Alternatively, apply the entire deployment file
                            # sed -i "s|image:.*|image: $DOCKER_REGISTRY/$APP_NAME:latest|" deployment.yaml
                            # kubectl apply -f deployment.yaml
                            
                            echo '🚀 Deployment updated with new image'
                        """
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh """
                        echo '🔍 Checking deployment status...'
                        
                        # Wait for rollout to complete
                        kubectl rollout status deployment/nodejs-app -n $K8S_NAMESPACE --timeout=300s
                        
                        echo '📊 Checking pods...'
                        kubectl get pods -n $K8S_NAMESPACE -l app=nodejs-app
                        
                        echo '🌐 Checking services...'
                        kubectl get svc -n $K8S_NAMESPACE
                        
                        echo '✅ Verification complete'
                    """
                }
            }
        }
        
        stage('Get Application URL') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        // Wait for LoadBalancer to get an IP
                        sh """
                            echo '⏳ Waiting for LoadBalancer to be ready...'
                            timeout 300s bash -c 'until kubectl get svc nodejs-app -n $K8S_NAMESPACE -o jsonpath="{.status.loadBalancer.ingress[0].hostname}" | grep -q "."; do sleep 10; done'
                        """
                        
                        // Get the LoadBalancer URL
                        def lbUrl = sh(
                            script: "kubectl get svc nodejs-app -n $K8S_NAMESPACE -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'",
                            returnStdout: true
                        ).trim()
                        
                        echo "🌐 Application URL: http://${lbUrl}"
                        
                        // Test the application
                        sh """
                            echo '🧪 Testing application...'
                            sleep 30
                            curl -f http://${lbUrl} || echo 'Application might still be starting...'
                        """
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo '🎉 Pipeline completed successfully!'
            emailext (
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """
                The pipeline completed successfully!
                
                Application URL: http://${lbUrl}
                Build: ${env.BUILD_URL}
                """,
                to: "admin@example.com"
            )
        }
        failure {
            echo '❌ Pipeline failed!'
            emailext (
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """
                The pipeline failed!
                
                Please check: ${env.BUILD_URL}
                """,
                to: "admin@example.com"
            )
        }
        always {
            echo '🧹 Cleaning up workspace...'
            cleanWs()
        }
    }
}
