
pipeline {
    agent any

    options {
        timestamps()
        skipDefaultCheckout(true)
    }

    environment {
        AWS_REGION     = 'ap-south-1'
        AWS_ACCOUNT_ID = '740778014300'
        EKS_CLUSTER    = 'nutriflow-eks'
        K8S_NAMESPACE  = 'nutriflow'
        ECR_REGISTRY   = '740778014300.dkr.ecr.ap-south-1.amazonaws.com'
        BACKEND_IMAGE  = 'nutriflow-backend'
        FRONTEND_IMAGE = 'nutriflow-frontend'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()
                }
                echo "Building images with tag: ${env.IMAGE_TAG}"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube-NutriFlow') {
                        sh(
                            label: 'Run SonarQube Scanner',
                            script: "${scannerHome}/bin/sonar-scanner"
                        )
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    set -e
                    docker build -t "$ECR_REGISTRY/$BACKEND_IMAGE:$IMAGE_TAG" ./backend
                    docker build -t "$ECR_REGISTRY/$FRONTEND_IMAGE:$IMAGE_TAG" ./frontend
                '''
            }
        }

        stage('Push Images to ECR') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-credentials',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        set +x
                        set -e
                        export AWS_DEFAULT_REGION="$AWS_REGION"

                        aws sts get-caller-identity

                        aws ecr get-login-password --region "$AWS_REGION" |
                          docker login --username AWS --password-stdin "$ECR_REGISTRY"

                        docker push "$ECR_REGISTRY/$BACKEND_IMAGE:$IMAGE_TAG"
                        docker push "$ECR_REGISTRY/$FRONTEND_IMAGE:$IMAGE_TAG"
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-credentials',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        set +x
                        set -e
                        export AWS_DEFAULT_REGION="$AWS_REGION"
                        export KUBECONFIG="$WORKSPACE/.kubeconfig"

                        aws eks update-kubeconfig \
                          --region "$AWS_REGION" \
                          --name "$EKS_CLUSTER" \
                          --kubeconfig "$KUBECONFIG"

                        kubectl -n "$K8S_NAMESPACE" set image \
                          deployment/backend \
                          backend="$ECR_REGISTRY/$BACKEND_IMAGE:$IMAGE_TAG"

                        kubectl -n "$K8S_NAMESPACE" set image \
                          deployment/frontend \
                          frontend="$ECR_REGISTRY/$FRONTEND_IMAGE:$IMAGE_TAG"

                        kubectl rollout status deployment/backend \
                          -n "$K8S_NAMESPACE" --timeout=300s

                        kubectl rollout status deployment/frontend \
                          -n "$K8S_NAMESPACE" --timeout=300s

                        kubectl get pods -n "$K8S_NAMESPACE"
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                rm -f "$WORKSPACE/.kubeconfig"
                docker logout "$ECR_REGISTRY" >/dev/null 2>&1 || true
            '''
        }
    }
}

