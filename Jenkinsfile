pipeline {
    agent any

    environment {
        DOCKERHUB_NAMESPACE = 'k0ola'
        IMAGE_NAME = "${DOCKERHUB_NAMESPACE}/backend-tp-note-pipeline"
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install') {
            steps {
                sh 'npm ci'
                sh 'npx prisma generate'
            }
        }

        stage('Unit tests and coverage') {
            steps {
                sh 'npm run test:coverage'
            }
            post {
                always {
                    junit 'reports/junit.xml'
                    archiveArtifacts artifacts: 'coverage/**', allowEmptyArchive: true
                }
            }
        }

        stage('Code quality (SonarQube)') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'sonar-scanner'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Docker build') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG -t $IMAGE_NAME:latest .'
            }
        }

        stage('Security scan and SBOM (Trivy)') {
            steps {
                sh 'trivy image --exit-code 0 --severity HIGH,CRITICAL --format table $IMAGE_NAME:$IMAGE_TAG'
                sh 'trivy image --format spdx-json --output sbom-spdx.json $IMAGE_NAME:$IMAGE_TAG'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'sbom-spdx.json', fingerprint: true, allowEmptyArchive: true
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
                    sh 'echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin'
                    sh 'docker push $IMAGE_NAME:$IMAGE_TAG'
                    sh 'docker push $IMAGE_NAME:latest'
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
        cleanup {
            cleanWs()
        }
    }
}
