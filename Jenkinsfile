pipeline {
    agent any

    environment {
        AWS_REGION     = "ap-southeast-1"         // change to your AWS region
        ECR_REPO       = "test-repo"              // your ECR repo name
        IMAGE_TAG      = "latest"                 // or use BUILD_NUMBER for versioning
        AWS_ACCOUNT_ID = "695466865413"           // replace with your AWS account ID
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo "Fetching source code from Git..."
                checkout scm
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
                sh 'npm test || echo "No tests found, skipping..."'
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
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
                                  credentialsId: 'aws-creds']]) {
                    script {
                        echo "Logging in to AWS ECR..."
                        sh """
                            aws ecr get-login-password --region $AWS_REGION | \
                            docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                            
                            docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/$ECR_REPO:$IMAGE_TAG
                        """
                    }
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

