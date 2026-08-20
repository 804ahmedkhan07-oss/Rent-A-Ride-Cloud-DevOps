pipeline {
    agent any

    environment {
        DOCKER_USERNAME = "ahmedmateen07"
        DOCKER_CREDS_ID = "dockerhub-creds"
        AWS_CREDS_ID    = "aws-creds"
        APP_NAME        = "rent-a-ride"
    }

    stages {
        stage('1. Checkout Code') {
            steps {
                echo 'Fetching latest code from GitHub'
                checkout scm
            }
        }

        stage('2. Build & Push Images') {
            steps {
                echo 'Building images and pushing to Docker Hub'
                withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh 'docker compose build'
                    sh 'docker compose push || true'
                }
            }
        }

        stage('3. Deploy Services') {
            steps {
                echo 'Deploying all services using Docker Compose'
                sh 'docker compose down || true'
                sh 'docker compose up -d --build'
            }
        }
    }

    post {
        always {
            echo 'Performing resource cleanup'
            sh 'docker logout || true'
            sh 'docker image prune -f'
        }
        success {
            echo 'Pipeline completed successfully'
        }
    }
}
