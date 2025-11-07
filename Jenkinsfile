pipeline {
    agent any
    
    environment {
        // AWS Configuration
        AWS_ACCOUNT_ID = '843559766730'
        AWS_REGION = 'us-east-1'
        ECR_REPO_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/my-app"
        EKS_CLUSTER_NAME = 'demo-cluster'
        
        // Application Configuration
        APP_NAME = 'my-app'
        K8S_NAMESPACE = 'default'
        DOCKER_IMAGE = "${ECR_REPO_URI}:${BUILD_NUMBER}"
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }
    
    parameters {
        choice(
            name: 'DEPLOY_ENVIRONMENT',
            choices: ['dev', 'staging', 'production'],
            description: 'Select deployment environment'
        )
        booleanParam(
            name: 'RUN_TESTS',
            defaultValue: true,
            description: 'Run tests before deployment'
        )
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
                script {
                    echo "Building commit: ${GIT_COMMIT}"
                    echo "Branch: ${GIT_BRANCH}"
                }
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh """
                    echo "Installing Node.js dependencies..."
                    npm install
                """
            }
        }
        
        stage('Run Tests') {
            when {
                expression { params.RUN_TESTS == true }
            }
            steps {
                sh """
                    echo "Running tests..."
                    npm test
                """
            }
            post {
                always {
                    junit 'reports/**/*.xml'
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image: ${DOCKER_IMAGE}"
                    sh """
                        docker build -t ${DOCKER_IMAGE} .
                        docker tag ${DOCKER_IMAGE} ${ECR_REPO_URI}:latest
                    """
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                script {
                    echo "Running security scan..."
                    // Uncomment if you have security scanners
                    // sh "trivy image ${DOCKER_IMAGE}"
                }
            }
        }
        
        stage('Push to ECR') {
            steps {
                script {
                    echo "Logging into ECR using IAM Role..."
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    """
                    
                    echo "Pushing images to ECR..."
                    sh """
                        docker push ${DOCKER_IMAGE}
                        docker push ${ECR_REPO_URI}:latest
                    """
                    
                    echo "✅ Images pushed successfully!"
                }
            }
        }
        
        stage('Verify ECR Push') {
            steps {
                script {
                    echo "Verifying images in ECR..."
                    sh """
                        aws ecr list-images \
                            --repository-name my-app \
                            --region ${AWS_REGION} \
                            --query 'imageIds[].imageTag' \
                            --output table
                    """
                }
            }
        }
        
        stage('Deploy to EKS') {
            when {
                expression { 
                    params.DEPLOY_ENVIRONMENT != null && 
                    env.EKS_CLUSTER_NAME != 'your-eks-cluster-name' 
                }
            }
            steps {
                script {
                    echo "Deploying to ${params.DEPLOY_ENVIRONMENT} environment..."
                    
                    // Configure kubectl for EKS cluster
                    sh """
                        aws eks update-kubeconfig \
                            --region ${AWS_REGION} \
                            --name ${EKS_CLUSTER_NAME}
                    """
                    
                    // Create namespace if it doesn't exist
                    sh """
                        kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                    """
                    
                    // Deploy application
                    sh """
                        kubectl set image deployment/${APP_NAME} \
                            ${APP_NAME}=${DOCKER_IMAGE} \
                            -n ${K8S_NAMESPACE} \
                            --record
                    """
                    
                    // Wait for rollout to complete
                    sh """
                        kubectl rollout status deployment/${APP_NAME} \
                            -n ${K8S_NAMESPACE} \
                            --timeout=600s
                    """
                    
                    echo "✅ Deployment completed successfully!"
                }
            }
        }
        
        stage('Smoke Test') {
            when {
                expression { params.DEPLOY_ENVIRONMENT != null }
            }
            steps {
                script {
                    echo "Running smoke tests..."
                    // Add your smoke test commands here
                    sh """
                        echo "Smoke tests would run here..."
                        kubectl get svc ${APP_NAME}-service -n ${K8S_NAMESPACE} || echo "No service found"
                    """
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "Pipeline execution completed: ${currentBuild.currentResult}"
                // Clean up Docker images to save space
                sh """
                    docker rmi ${DOCKER_IMAGE} || true
                    docker rmi ${ECR_REPO_URI}:latest || true
                    docker system prune -f || true
                """
            }
        }
        success {
            script {
                echo "🎉 SUCCESS! Pipeline completed using IAM Role - No AWS credentials stored!"
                
                // Send success notification (uncomment if needed)
                /*
                emailext (
                    subject: "SUCCESS: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                    body: """
                    Pipeline execution successful!
                    
                    Job: ${env.JOB_NAME}
                    Build: ${env.BUILD_NUMBER}
                    Environment: ${params.DEPLOY_ENVIRONMENT}
                    Image: ${DOCKER_IMAGE}
                    """,
                    to: "devops@yourcompany.com"
                )
                */
            }
        }
        failure {
            script {
                echo "❌ Pipeline failed!"
                // Send failure notification if needed
            }
        }
    }
}
