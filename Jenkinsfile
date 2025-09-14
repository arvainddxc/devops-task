pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = "695466865413"
        AWS_REGION     = "ap-southeast-1"
        ECR_REPO       = "test-repo"
        IMAGE_TAG      = "latest"
        CLUSTER_NAME   = "my-cluster"   // change to your cluster name
        K8S_NAMESPACE  = "test"          // change if using custom namespace
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'dev', url: 'https://github.com/arvainddxc/devops-task.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'npm test'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t $ECR_REPO:$IMAGE_TAG .
                    docker tag $ECR_REPO:$IMAGE_TAG ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/$ECR_REPO:$IMAGE_TAG
                """
            }
        }

        stage('Push to ECR') {
            steps {
                sh """
                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/$ECR_REPO:$IMAGE_TAG
                """
            }
        }

        stage('Update Kubeconfig') {
            steps {
                sh """
                    aws eks --region $AWS_REGION update-kubeconfig --name $CLUSTER_NAME
                """
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh """
                    kubectl set image deployment/test-app test-app=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/$ECR_REPO:$IMAGE_TAG -n $K8S_NAMESPACE || \
                    kubectl apply -f k8s/
                """
            }
        }
    }
}
