pipeline {
    agent any

    environment {
        AWS_REGION = "us-east-1"
        ECR_REGISTRY = "723619976108.dkr.ecr.us-east-1.amazonaws.com"
        ECR_REPO = "flask-app-repo"
        IMAGE_TAG = "v${BUILD_NUMBER}"
        KUBECONFIG_PATH = "/var/lib/jenkins/.kube/config"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git(
                    branch: 'main',
                    url: 'https://github.com/user45-desgin/aws-eks-prometheus-grafana-monitoring.git'
                )
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    echo "Building Docker image..."
                    docker build -t ${ECR_REPO}:${IMAGE_TAG} ./flask-app
                """
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                  secretKeyVariable: 'AWS_SECRET_ACCESS_KEY',
                                  credentialsId: 'AWS-CREDENTIALS']]) {
                    sh """
                        echo "Logging into Amazon ECR..."
                        aws ecr get-login-password --region ${AWS_REGION} \
                        | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    """
                }
            }
        }

        stage('Tag & Push Image to ECR') {
            steps {
                sh """
                    echo "Pushing image to ECR..."
                    docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                """
            }
        }

        stage('Update Helm Chart') {
            steps {
                sh """
                    echo "Updating Helm chart..."
                    sed -i 's|repository: .*|repository: ${ECR_REGISTRY}/${ECR_REPO}|' helm-chart/flask-chart/flask-chart/values.yaml
                    sed -i 's|tag: .*|tag: "${IMAGE_TAG}"|' helm-chart/flask-chart/flask-chart/values.yaml
                """
            }
        }

        stage('Deploy to EKS Using Helm') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                  secretKeyVariable: 'AWS_SECRET_ACCESS_KEY',
                                  credentialsId: 'AWS-CREDENTIALS']]) {
                    sh """
                        echo "Using existing kubeconfig..."
                        export KUBECONFIG=${KUBECONFIG_PATH}

                        echo "Deploying application to EKS..."
                        helm upgrade --install flask-release helm-chart/flask-chart/flask-chart \
                          --namespace flask-app --create-namespace
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh """
                    echo "Checking deployment rollout..."
                    export KUBECONFIG=${KUBECONFIG_PATH}
                    kubectl rollout status deployment/flask-release-flask-chart -n flask-app
                """
            }
        }
    }
}
