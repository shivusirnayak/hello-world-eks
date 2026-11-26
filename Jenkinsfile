
pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = '194398118151'
        AWS_REGION     = 'eu-west-1'

        ECR_REPOSITORY = 'hello-world'
        EKS_CLUSTER    = 'REPLACE_WITH_YOUR_EKS_CLUSTER_NAME'

        NAMESPACE      = 'hello-world'
        APP_NAME       = 'hello-world'
    }

    stages {

        stage('Checkout GitHub') {
            steps {
                checkout scm
            }
        }

        stage('Create ECR Repository') {
            steps {
                sh '''
                    aws ecr describe-repositories \
                      --repository-names $ECR_REPOSITORY \
                      --region $AWS_REGION >/dev/null 2>&1 || \
                    aws ecr create-repository \
                      --repository-name $ECR_REPOSITORY \
                      --region $AWS_REGION
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = "${BUILD_NUMBER}"
                    env.ECR_REGISTRY =
                        "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                }

                sh '''
                    docker build \
                      -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password \
                      --region $AWS_REGION | \
                    docker login \
                      --username AWS \
                      --password-stdin $ECR_REGISTRY

                    docker push \
                      $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
                '''
            }
        }

        stage('Connect to EKS') {
            steps {
                sh '''
                    aws eks update-kubeconfig \
                      --region $AWS_REGION \
                      --name $EKS_CLUSTER

                    kubectl get nodes
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    kubectl create namespace $NAMESPACE \
                      --dry-run=client -o yaml | kubectl apply -f -

                    kubectl apply -f k8s/

                    kubectl set image \
                      deployment/$APP_NAME \
                      $APP_NAME=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
                      -n $NAMESPACE

                    kubectl rollout status \
                      deployment/$APP_NAME \
                      -n $NAMESPACE \
                      --timeout=180s
                '''
            }
        }

        stage('Get Application URL') {
            steps {
                sh '''
                    echo "Application Ingress:"
                    kubectl get ingress -n $NAMESPACE

                    echo "Application Pods:"
                    kubectl get pods -n $NAMESPACE
                '''
            }
        }
    }

    post {
        success {
            echo 'Hello World application deployed successfully!'
        }

        failure {
            echo 'Hello World deployment failed. Check Jenkins console output.'
        }
    }
}

