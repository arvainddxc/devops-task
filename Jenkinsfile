pipeline {
    agent any

    environment {
        AWS_REGION     = "ap-southeast-1"
        AWS_ACCOUNT_ID = "695466865413"
        ECR_REPO       = "test-repo"
        IMAGE_TAG      = "v1"
        EKS_CLUSTER    = "my-cluster"
        APP_NAME       = "test-app"
        JUMP_HOST      = "13.213.70.212"   // easier to reuse
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo "📥 Fetching source code..."
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
                echo "🐳 Building Docker image..."
                sh """
                    docker build -t $ECR_REPO:$IMAGE_TAG .
                    docker tag $ECR_REPO:$IMAGE_TAG ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/$ECR_REPO:$IMAGE_TAG
                """
            }
        }

        stage('Login to ECR & Push Image') {
            steps {
                echo "🔑 Logging in to Amazon ECR..."
                sh """
                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                    echo "⬆️ Pushing Docker image..."
                    docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/$ECR_REPO:$IMAGE_TAG
                """
            }
        }

        stage('Deploy to EKS via Jump Host') {
            steps {
                sshagent(['eks-ssh']) {
                    sh """
                        echo "🚀 Connecting to Jump Host..."
                        ssh -o StrictHostKeyChecking=no ubuntu@$JUMP_HOST "mkdir -p /home/ubuntu/k8s"

                        echo "📂 Copying Kubernetes manifests..."
                        scp -o StrictHostKeyChecking=no -r k8s/deployment.yaml k8s/service.yaml ubuntu@$JUMP_HOST:/home/ubuntu/k8s/

                        echo "⚡ Running deployment on Jump Host..."
                        ssh -o StrictHostKeyChecking=no ubuntu@$JUMP_HOST bash -c "'
                            set -e

                            echo ✅ Verifying AWS identity...
                            aws sts get-caller-identity

                            echo ✅ Updating kubeconfig for cluster: ${EKS_CLUSTER}...
                            aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}

                            cd /home/ubuntu/k8s
                            echo 📝 Updating deployment.yaml with new image...
                            sed -i \\"s|image:.*|image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}|g\\" deployment.yaml

                            echo 📦 Applying manifests...
                            kubectl apply -f deployment.yaml --validate=false
                            kubectl apply -f service.yaml --validate=false

                            echo ⏳ Waiting for rollout...
                            kubectl rollout status deployment/${APP_NAME} || true

                            echo ✅ Checking pods...
                            kubectl get pods -o wide
                        '"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline executed successfully!"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}
