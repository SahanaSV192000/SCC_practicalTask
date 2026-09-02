pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }
    environment {
        AWS_REGION = 'us-east-1'
        ECR_REGISTRY = '656446902704.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPOSITORY = 'scc-practical-task'
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
        K8S_NAMESPACE = 'scc-practical-task'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            steps {
                sh 'mvn -B clean test'
            }

            post {
                always {
                    junit(
                        testResults: 'target/surefire-reports/*.xml',
                        allowEmptyResults: true
                    )
                }
            }
        }

        stage('Package') {
            steps {
                sh 'mvn -B package -DskipTests'
                archiveArtifacts(
                    artifacts: 'target/*.jar',
                    fingerprint: true
                )
            }
        }
        
        stage('Docker Build') {
            steps {
                sh "docker build --pull --no-cache -t scc-practical-task:${BUILD_NUMBER} ."
                sh 'docker image inspect scc-practical-task:${BUILD_NUMBER} > /dev/null'
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                    trivy image \
                        --exit-code 1 \
                        --severity HIGH,CRITICAL \
                        --ignore-unfixed \
                        scc-practical-task:${BUILD_NUMBER}
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

        stage('Login, Tag and Push to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region "$AWS_REGION" | \
                    docker login --username AWS --password-stdin "$ECR_REGISTRY"

                    docker tag \
                    scc-practical-task:${BUILD_NUMBER} \
                    "$ECR_REGISTRY/$ECR_REPOSITORY:${BUILD_NUMBER}"

                    docker push \
                    "$ECR_REGISTRY/$ECR_REPOSITORY:${BUILD_NUMBER}"
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    set -eu

                    kubectl create namespace "$K8S_NAMESPACE" \
                    --dry-run=client -o yaml | kubectl apply -f -

                    ECR_PASSWORD="$(aws ecr get-login-password --region "$AWS_REGION")"

                    kubectl create secret docker-registry ecr-registry \
                    --namespace "$K8S_NAMESPACE" \
                    --docker-server="$ECR_REGISTRY" \
                    --docker-username=AWS \
                    --docker-password="$ECR_PASSWORD" \
                    --dry-run=client -o yaml | kubectl apply -f -

                    sed "s|IMAGE_PLACEHOLDER|$ECR_REGISTRY/$ECR_REPOSITORY:$BUILD_NUMBER|g" \
                    k8s/app.yaml | kubectl apply -f -

                    kubectl rollout status deployment/scc-practical-task \
                    --namespace "$K8S_NAMESPACE" \
                    --timeout=180s
                '''
            }
        }
    }

    post {
        success {
            echo 'Build and test completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the console output.'
        }
    }
}