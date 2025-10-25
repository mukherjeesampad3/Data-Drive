pipeline {
    agent any

    environment {
        IMAGE_NAME = "data-drive-container"
        EC2_HOST = "13.234.113.90"
    }

    stages {
        stage('Checkout') {
            steps {
                echo "📦 Cloning repository..."
                git branch: 'main', url: 'https://github.com/mukherjeesampad3/Data-Drive.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🐳 Building Docker image..."
                sh '''
                    set -e
                    echo "🧩 Checking Docker access..."
                    docker info > /dev/null
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

                        echo "📦 Tagging image..."
                        docker tag ${IMAGE_NAME}:latest "$DOCKER_USER/${IMAGE_NAME}:latest"

                        echo "🚀 Pushing image to Docker Hub..."
                        docker push "$DOCKER_USER/${IMAGE_NAME}:latest"

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                echo "🚀 Deploying to EC2..."
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
                    usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')
                ]) {
                    sh '''
                        set -e
                        chmod 600 "$SSH_KEY"

                        echo "🔗 Connecting to EC2 host: ${EC2_HOST} ..."
                        ssh -o StrictHostKeyChecking=no -i "$SSH_KEY" "$SSH_USER"@"${EC2_HOST}" bash -s <<'EOF'
                            set -e
                            echo "🔑 Logging in to Docker Hub..."
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                            echo "📥 Pulling latest image..."
                            docker pull "$DOCKER_USER/${IMAGE_NAME}:latest"

                            echo "🧹 Removing old container if exists..."
                            docker rm -f data-drive 2>/dev/null || true

                            echo "🚀 Starting new container..."
                            docker run -d \
                                --name data-drive \
                                --env-file /root/Data-Drive/.env \
                                -p 3000:3000 \
                                --restart unless-stopped \
                                "$DOCKER_USER/${IMAGE_NAME}:latest"

                            echo "🔍 Checking container status..."
                            docker ps | grep data-drive || (echo '⚠️ Container not found!' && exit 1)

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
            echo "🌐 Application should be running at http://${EC2_HOST}:3000"
        }
        failure {
            echo "❌ Deployment failed!"
        }
        always {
            echo "🧹 Cleaning up local Docker resources..."
            sh '''
                docker logout 2>/dev/null || true
            '''
        }
    }
}
