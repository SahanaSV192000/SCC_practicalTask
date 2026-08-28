pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
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
                sh 'docker build --pull -t scc-practical-task:${BUILD_NUMBER} .'
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