pipeline {
    agent any

    environment {
        AWS_REGION     = "ap-southeast-1"        // your AWS region
        ECR_REPO       = "test-repo"             // your ECR repo name
        IMAGE_TAG      = "v1"                    // or use BUILD_NUMBER for versioning
        AWS_ACCOUNT_ID = "695466865413"          // your AWS account ID
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo "Fetching source code from GitHub..."
                git branch: 'dev',
                    url: 'https://github.com/arvainddxc/devops-task.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                echo "Installing Node.js dependencies..."
                sh 'npm install'
            }
        }

        stage('Run Tests') {
            steps {
                echo "Running tests..."
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
                    echo "Building Docker image..."
                    sh """
                        docker build -t $ECR_REPO:$IMAGE_TAG .
                        docker tag $ECR_REPO:$IMAGE_TAG ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/$ECR_REPO:$IMAGE_TAG
                    """
                }
            }
        }

        stage('Login to ECR & Push Image') {
            steps {
                // Use AWS credentials stored in Jenkins
                withCredentials([usernamePassword(credentialsId: 'aws-creds',
                                                  usernameVariable: 'AWS_ACCESS_KEY_ID',
                                                  passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh """
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set default.region $AWS_REGION
                        
                        aws ecr get-login-password --region $AWS_REGION | \
                        docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                        
                        docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/$ECR_REPO:$IMAGE_TAG
                    """
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    echo "Deploying to EKS..."
                    sh """
                        aws eks update-kubeconfig --region $AWS_REGION --name my-eks-cluster
                        kubectl set image deployment/my-app my-app=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/$ECR_REPO:$IMAGE_TAG -n default
                        kubectl rollout status deployment/my-app 
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
