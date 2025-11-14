pipeline {
    agent any

    environment {
        AWS_REGION = "us-east-1"
        ECR_REGISTRY = "723619976108.dkr.ecr.us-east-1.amazonaws.com"
        ECR_REPO = "flask-app-repo"
        IMAGE_TAG = "v${BUILD_NUMBER}"
        KUBECONFIG = credentials('KUBE_CONFIG')
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/user45-desgin/aws-eks-prometheus-grafana-monitoring.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                        echo "Building Docker Image..."
                        docker build -t ${ECR_REPO}:${IMAGE_TAG} ./flask-app
                    """
                }
            }
        }

        stage('Login to Amazon ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'AWS-CREDENTIALS']]) {
                    sh """
                        echo "Logging in to Amazon ECR..."
                        aws ecr get-login-password --region ${AWS_REGION} \
                        | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    """
                }
            }
        }

        stage('Tag & Push Image to ECR') {
            steps {
                sh """
                    echo "Tagging and pushing image to ECR..."
                    docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                """
            }
        }

        stage('Update Helm Chart With New Image') {
            steps {
                sh """
                    echo "Updating Helm chart with new image..."
                    sed -i 's|repository: .*|repository: ${ECR_REGISTRY}/${ECR_REPO}|' helm-chart/flask-chart/values.yaml
                    sed -i 's|tag: .*|tag: "${IMAGE_TAG}"|' helm-chart/flask-chart/values.yaml
                """
            }
        }

        stage('Deploy to EKS Using Helm') {
            steps {
                script {
                    withAWS(region: "${AWS_REGION}", credentials: 'AWS-CREDENTIALS') {
                        sh """
                            echo "Deploying to EKS using Helm..."
                            aws eks update-kubeconfig --region ${AWS_REGION} --name flask-cluster
                            helm upgrade --install flask-release helm-chart/flask-chart \
                              --namespace flask-app --create-namespace
                        """
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh """
                    echo "Verifying Deployment..."
                    kubectl rollout status deployment/flask-release-flask-chart -n flask-app
                """
            }
        }
    }
}
