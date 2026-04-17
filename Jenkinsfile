pipeline {
    agent any

    environment {
        AWS_REGION        = "us-east-1"
        AWS_ACCOUNT_ID    = "439481669447"
        ECR_REPO          = "myapp"
        IMAGE_TAG         = "${BUILD_NUMBER}"
        ECR_URI           = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
        CLUSTER_NAME      = "myapp-eks"
    }

    stages {

        stage('Configure AWS') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id',     variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh """
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set region $AWS_REGION
                    """
                }
            }
        }

        stage('Terraform VPC') {
            steps {
                dir('terraform/vpc') {
                    sh """
                        terraform init
                        terraform plan -out=tfplan
                        terraform apply -auto-approve tfplan
                    """
                }
            }
        }

        stage('Terraform EKS') {
            steps {
                dir('terraform/eks') {
                    sh """
                        terraform init
                        terraform plan -out=tfplan
                        terraform apply -auto-approve tfplan
                    """
                }
            }
        }

        stage('Create ECR Repo') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id',     variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh """
                        aws ecr describe-repositories --repository-names $ECR_REPO --region $AWS_REGION || \
                        aws ecr create-repository --repository-name $ECR_REPO --region $AWS_REGION
                    """
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id',     variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    dir('app') {
                        sh """
                            aws ecr get-login-password --region $AWS_REGION | \
                            docker login --username AWS --password-stdin $ECR_URI

                            docker build -t $ECR_REPO:$IMAGE_TAG .
                            docker tag $ECR_REPO:$IMAGE_TAG $ECR_URI:$IMAGE_TAG
                            docker tag $ECR_REPO:$IMAGE_TAG $ECR_URI:latest
                            docker push $ECR_URI:$IMAGE_TAG
                            docker push $ECR_URI:latest
                        """
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id',     variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh """
                        aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                        kubectl rollout status deployment/myapp
                    """
                }
            }
        }
    }

    post {
        success { echo "Pipeline completed successfully!" }
        failure { echo "Pipeline failed. Check logs above." }
    }
}

