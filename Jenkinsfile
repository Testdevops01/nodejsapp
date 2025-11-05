pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID = credentials('aws-account-id')
        AWS_ACCESS_KEY_ID = credentials('aws-access-key')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
        DOCKER_REGISTRY = 'anusha987'
        APP_NAME = 'nodejsapp-9.0'
        EKS_CLUSTER_NAME = 'nodejs-app-cluster'
        AWS_REGION = 'us-west-2'
        GIT_REPO = 'https://github.com/Testdevops01/nodejsapp'
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: "${GIT_REPO}"
            }
        }
        
        stage('Create Application Files') {
            steps {
                sh '''
                mkdir -p nodejs-app
                cat > nodejs-app/package.json << 'EOF'
                {
                    "name": "docker_web_app",
                    "version": "1.0.0",
                    "description": "Node.js on Docker",
                    "author": "First Last <first.last@example.com>",
                    "main": "server.js",
                    "scripts": {
                      "start": "node server.js"
                    },
                    "dependencies": {
                      "express": "^4.16.1"
                    }
                }
                EOF
                
                cat > nodejs-app/server.js << 'EOF'
                'use strict';
                
                const express = require('express');
                
                // Constants
                const PORT = 3000;
                const HOST = '0.0.0.0';
                
                // App
                const app = express();
                app.get('/', (req, res) => {
                  res.send('Hello World');
                });
                
                app.listen(PORT, HOST);
                console.log(\`Running on http://\${HOST}:\${PORT}\`);
                EOF
                
                cat > nodejs-app/Dockerfile << 'EOF'
                FROM node:14-alpine
                WORKDIR /usr/src/app
                COPY package*.json ./
                RUN npm install
                COPY . .
                EXPOSE 3000
                CMD [ "node", "server.js" ]
                EOF
                '''
            }
        }
        
        stage('Setup AWS CLI and EKSCTL') {
            steps {
                sh '''
                # Install AWS CLI
                curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                unzip awscliv2.zip
                sudo ./aws/install
                
                # Install eksctl
                curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
                sudo mv /tmp/eksctl /usr/local/bin
                
                # Install kubectl
                curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
                
                # Configure AWS CLI
                aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}
                aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}
                aws configure set default.region ${AWS_REGION}
                '''
            }
        }
        
        stage('Create EKS Cluster') {
            steps {
                sh """
                # Create EKS cluster configuration
                cat > eks-cluster.yaml << EOF
                apiVersion: eksctl.io/v1alpha5
                kind: ClusterConfig
                metadata:
                  name: ${EKS_CLUSTER_NAME}
                  region: ${AWS_REGION}
                  version: "1.28"
                nodeGroups:
                  - name: ng-1
                    instanceType: t3.medium
                    desiredCapacity: 2
                    minSize: 1
                    maxSize: 3
                    volumeSize: 20
                    ssh:
                      allow: true
                EOF
                
                # Create EKS cluster
                eksctl create cluster -f eks-cluster.yaml
                
                # Update kubeconfig
                aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}
                """
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh """
                cd nodejs-app
                docker build -t ${DOCKER_REGISTRY}/${APP_NAME}:latest .
                """
            }
        }
        
        stage('Push Docker Image') {
            steps {
                sh """
                # Login to Docker Hub (or your preferred registry)
                docker login -u ${DOCKER_REGISTRY} -p ${env.DOCKERHUB_CREDENTIALS}
                
                # Push image
                docker push ${DOCKER_REGISTRY}/${APP_NAME}:latest
                """
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                sh """
                # Create Kubernetes deployment manifest
                cat > k8s-deployment.yaml << EOF
                ---
                kind: Deployment
                apiVersion: apps/v1
                metadata:
                  name: nodejs-app
                  namespace: default
                  labels:
                    app: nodejs-app
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
                        image: "${DOCKER_REGISTRY}/${APP_NAME}:latest"
                        ports:
                          - containerPort: 3000
                        resources:
                          requests:
                            memory: "128Mi"
                            cpu: "100m"
                          limits:
                            memory: "256Mi"
                            cpu: "200m"
                ---
                apiVersion: v1
                kind: Service
                metadata:
                  name: nodejs-app
                  namespace: default
                spec:
                  selector:
                    app: nodejs-app
                  type: LoadBalancer
                  ports:
                  - name: http
                    targetPort: 3000
                    port: 80
                EOF
                
                # Deploy to EKS
                kubectl apply -f k8s-deployment.yaml
                
                # Wait for deployment to be ready
                kubectl rollout status deployment/nodejs-app --timeout=300s
                """
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sh """
                # Check pods
                kubectl get pods
                
                # Check services
                kubectl get svc
                
                # Check deployment
                kubectl get deployment nodejs-app
                """
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline execution completed'
            // Cleanup workspace if needed
            // deleteDir()
        }
        success {
            echo 'Application deployed successfully to EKS!'
            sh '''
            # Get LoadBalancer URL
            kubectl get svc nodejs-app -o jsonpath="{.status.loadBalancer.ingress[0].hostname}"
            '''
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
