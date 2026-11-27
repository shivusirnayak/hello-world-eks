
pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = '033781183938'
        AWS_REGION     = 'ap-southeast-2'

        ECR_REPOSITORY = 'hello-world'
        EKS_CLUSTER    = 'test-cluster'

        NAMESPACE      = 'test'
        APP_NAME       = 'hello-world'

        ECR_REGISTRY   = '033781183938.dkr.ecr.ap-southeast-2.amazonaws.com'
    }

    stages {

        stage('Checkout GitHub') {
            steps {
                checkout scm
            }
        }

        stage('Validate Prerequisites') {
            steps {
                sh '''
                    echo "Checking AWS identity..."
                    aws sts get-caller-identity

                    echo "Checking Docker..."
                    docker --version

                    echo "Checking kubectl..."
                    kubectl version --client

                    echo "Checking AWS CLI..."
                    aws --version
                '''
            }
        }

        stage('Create ECR Repository') {
            steps {
                sh '''
                    aws ecr describe-repositories \
                      --repository-names "$ECR_REPOSITORY" \
                      --region "$AWS_REGION" >/dev/null 2>&1 || \
                    aws ecr create-repository \
                      --repository-name "$ECR_REPOSITORY" \
                      --region "$AWS_REGION"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = "${BUILD_NUMBER}"
                }

                sh '''
                    docker build \
                      -t "$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" .
                '''
            }
        }

        stage('Login and Push to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password \
                      --region "$AWS_REGION" | \
                    docker login \
                      --username AWS \
                      --password-stdin "$ECR_REGISTRY"

                    docker push \
                      "$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"
                '''
            }
        }

        
stage('Connect to EKS') {
    steps {
        sh '''
            aws eks update-kubeconfig \
              --region ${AWS_REGION} \
              --name ${EKS_CLUSTER}

            echo "Kubernetes context:"
            kubectl config current-context

            echo "Checking namespace access:"
            kubectl get namespace ${NAMESPACE}
        '''
    }
}


        stage('Create Namespace') {
            steps {
                sh '''
                    kubectl create namespace "$NAMESPACE" \
                      --dry-run=client -o yaml | kubectl apply -f -
                '''
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                    echo "Applying Kubernetes manifests..."

                    kubectl apply -f k8s/ -n "$NAMESPACE"

                    echo "Updating application image..."

                    kubectl set image \
                      deployment/"$APP_NAME" \
                      "$APP_NAME=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" \
                      -n "$NAMESPACE"

                    echo "Waiting for deployment rollout..."

                    kubectl rollout status \
                      deployment/"$APP_NAME" \
                      -n "$NAMESPACE" \
                      --timeout=180s
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Deployment status:"
                    kubectl get deployment "$APP_NAME" -n "$NAMESPACE"

                    echo "Running pods:"
                    kubectl get pods -n "$NAMESPACE" -o wide

                    echo "Services:"
                    kubectl get svc -n "$NAMESPACE"

                    echo "Ingress:"
                    kubectl get ingress -n "$NAMESPACE"
                '''
            }
        }

        stage('Get Application URL') {
            steps {
                script {
                    def ingressHost = sh(
                        script: '''
                            kubectl get ingress -n "$NAMESPACE" \
                              -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}'
                        ''',
                        returnStdout: true
                    ).trim()

                    if (ingressHost) {
                        echo "Hello World URL: http://${ingressHost}"
                    } else {
                        echo "ALB hostname is not available yet."
                        echo "Check ingress status using kubectl get ingress -n ${NAMESPACE}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Hello World application deployed successfully!'
        }

        failure {
            echo 'Deployment failed. Check Jenkins Console Output.'
        }

        always {
            echo 'Jenkins pipeline execution completed.'
        }
    }
}
