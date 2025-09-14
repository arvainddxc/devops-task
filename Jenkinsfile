pipeline {
    agent any

    environment {
        AWS_REGION     = "ap-southeast-1"        // AWS region
        ECR_REPO       = "test-repo"             // ECR repo name
        IMAGE_TAG      = "v1"                    // or use BUILD_NUMBER
        AWS_ACCOUNT_ID = "695466865413"          // AWS account ID
        EKS_CLUSTER    = "my-cluster"            // your EKS cluster name
        APP_NAME       = "test-app"              // k8s Deployment name
        JUMP_HOST      = "13.213.70.212"         // Bastion/jump EC2 host for kubectl
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo "📦 Fetching source code..."
                git branch: 'dev',
                    url: 'https://github.com/arvainddxc/devops-task.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                echo "📦 Installing Node.js dependencies..."
                sh 'npm install'
            }
        }

        stage('Run Tests') {
            steps {
                echo "🧪 Running tests..."
                sh '''
                if npm run | grep -q "test"; then
                  npm test
                else
                  echo "⚠️ No test script found in package.json, skipping..."
                fi
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "🐳 Building Docker image..."
                    sh """
                        docker build -t $ECR_REPO:$IMAGE_TAG .
                        docker tag $ECR_REPO:$IMAGE_TAG ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/$ECR_REPO:$IMAGE_TAG
                    """
                }
            }
        }

        stage('Login & Push to ECR') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-creds',
                                                  usernameVariable: 'AWS_ACCESS_KEY_ID',
                                                  passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                        echo "🔐 Logging into AWS ECR..."
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set default.region $AWS_REGION
                        
                        aws ecr get-login-password --region $AWS_REGION | \
                        docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                        
                        echo "⬆️ Pushing Docker image to ECR..."
                        docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/$ECR_REPO:$IMAGE_TAG
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-creds',
                                                  usernameVariable: 'AWS_ACCESS_KEY_ID',
                                                  passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sshagent(['eks-ssh']) {
                        sh """
                            echo "🚀 Connecting to Jump Host $JUMP_HOST..."
                            ssh -o StrictHostKeyChecking=no ubuntu@$JUMP_HOST "mkdir -p /home/ubuntu/k8s"

                            echo "📂 Copying Kubernetes manifests..."
                            scp -o StrictHostKeyChecking=no -r k8s/* ubuntu@$JUMP_HOST:/home/ubuntu/k8s/

                            ssh -o StrictHostKeyChecking=no ubuntu@$JUMP_HOST << 'EOF'
                                set -e

                                export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                                export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                                export AWS_DEFAULT_REGION=$AWS_REGION

                                echo "✅ Verifying AWS identity..."
                                aws sts get-caller-identity

                                echo "✅ Updating kubeconfig for cluster: $EKS_CLUSTER..."
                                aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER

                                cd /home/ubuntu/k8s

                                echo "📝 Updating deployment.yaml with new image..."
                                sed -i "s|image:.*|image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}|g" deployment.yaml

                                echo "📦 Applying manifests..."
                                kubectl apply -f deployment.yaml --validate=false
                                kubectl apply -f service.yaml --validate=false

                                echo "⏳ Waiting for rollout of deployment $APP_NAME..."
                                kubectl rollout status deployment/$APP_NAME || true

                                echo "🔍 Checking pods..."
                                kubectl get pods -o wide
                            EOF
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline executed successfully! 🎉"
        }
        failure {
            echo "❌ Pipeline failed! Please check logs."
        }
    }
}
