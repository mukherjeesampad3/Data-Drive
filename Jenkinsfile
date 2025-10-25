pipeline {
    agent any

    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Git branch to build')
        string(name: 'APP_PORT', defaultValue: '3000', description: 'Port to expose the container')
    }

    environment {
        IMAGE_NAME = "data-drive-container"
        EC2_HOST = "13.234.113.90"
    }

    stages {
        stage('Checkout') {
            steps {
                echo "📦 Cloning repository from branch '${params.BRANCH_NAME}'..."
                git branch: "${params.BRANCH_NAME}", url: 'https://github.com/mukherjeesampad3/Data-Drive.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🐳 Building Docker image '${IMAGE_NAME}:latest'..."
                sh '''
                    set -e
                    docker build -t ${IMAGE_NAME}:latest .
                '''
            }
        }

        stage('Login & Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        set -e
                        echo "🔑 Logging in to Docker Hub..."
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        echo "📤 Tagging and pushing image..."
                        docker tag ${IMAGE_NAME}:latest "$DOCKER_USER/${IMAGE_NAME}:latest"
                        docker push "$DOCKER_USER/${IMAGE_NAME}:latest"

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                echo "🚀 Deploying to EC2 instance ${EC2_HOST}..."
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
                    usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')
                ]) {
                    sh '''
                        set -e
                        chmod 600 "$SSH_KEY"

                        ssh -o StrictHostKeyChecking=no -i "$SSH_KEY" "$SSH_USER@${EC2_HOST}" << EOF
                            set -e
                            echo "🔑 Logging in to Docker Hub..."
                            echo '$DOCKER_PASS' | docker login -u '$DOCKER_USER' --password-stdin

                            echo "📥 Pulling latest image..."
                            docker pull '$DOCKER_USER/${IMAGE_NAME}:latest'

                            echo "🧹 Removing old container if exists..."
                            docker stop data-drive || true
                            docker rm data-drive || true

                            echo "🚀 Starting new container..."
                            docker run -d -p ${APP_PORT}:${APP_PORT} --name data-drive --restart unless-stopped '$DOCKER_USER/${IMAGE_NAME}:latest'

                            echo "🔍 Verifying container status..."
                            docker ps | grep data-drive || echo '⚠️ Container not running!'

                            docker logout
                        EOF
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment successful!"
            echo "🌐 Application should be running at http://${EC2_HOST}:${params.APP_PORT}"
        }
        failure {
            echo "❌ Deployment failed!"
        }
        always {
            echo "🧹 Cleaning up local Docker resources..."
            sh '''
                docker logout || true
                docker image prune -f || true
            '''
        }
    }
}
