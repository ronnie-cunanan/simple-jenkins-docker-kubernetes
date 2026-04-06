pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-southeast-2'
        AWS_ACCOUNT_ID = "962047682202"
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_REPOSITORY = 'my-docker-repo'
        IMAGE_TAG = "${BUILD_NUMBER}"
        KUBECONFIG = "/home/jenkins/.kube/config"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scmGit(branches: [[name: '*/main']], extensions: [], 
                userRemoteConfigs: [[url: 'https://github.com/ronnie-cunanan/simple-jenkins-docker-kubernetes.git']])
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh '''
                        docker build -t ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG} .
                        docker tag ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest                        
                    '''
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    sh '''
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}
                        docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest
                    '''
                }
            }
        }

        stage('Verify kubectl Connectivity') {
            steps {
                sh """
                    set -e

                    echo "Testing kubectl installation..."
                    if ! kubectl version --client; then
                        echo "ERROR: kubectl not installed in Jenkins container"
                        exit 1
                    fi

                    echo "Testing Kubernetes API connectivity..."
                    if ! kubectl --kubeconfig=${KUBECONFIG} version; then
                        echo "ERROR: Cannot reach Kubernetes API server"
                        exit 1
                    fi

                    echo "Testing node access..."
                    if ! kubectl --kubeconfig=${KUBECONFIG} get nodes; then
                        echo "ERROR: Jenkins cannot list nodes (RBAC or network issue)"
                        exit 1
                    fi

                    echo "kubectl connectivity OK."
                """
            }
        }

        stage('Refresh ECR Secret') {
            steps {
                sh """
                # Get a fresh token and recreate the secret
                TOKEN=\$(aws ecr get-login-password --region ap-southeast-2)
                
                # Delete the old secret first (to ensure a fresh one)
                kubectl delete secret ecr-secret --ignore-not-found
                
                # Create the new secret
                kubectl create secret docker-registry ecr-secret \
                --docker-server=${ECR_REGISTRY} \
                --docker-username=AWS \
                --docker-password=\$TOKEN
                """
            }
        }

        stage('Deploy Base Kubernetes Resources') {
            steps {
                sh """
                kubectl apply -f k8s/deployment.yaml
                kubectl apply -f k8s/service.yaml
                """
            }
        }

        stage('Update Image in Kubernetes') {
            steps {
                sh """
                kubectl set image deployment/my-flask-app my-flask-app=${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                sh """
                kubectl rollout status deployment/my-flask-app
                kubectl get pods -o wide
                """
            }
        }
    }

    post {
        success {
            echo "Deployment successful: ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"
        }
        failure {
            echo "Build or deployment failed. Check stage logs."
        }
    }
}