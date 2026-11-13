pipeline {
    agent any

    environment {
        // DockerHub Configuration
        DOCKERHUB_CREDENTIALS = 'dockerhub-credentials' // Jenkins Credentials ID (Username with password)
        DOCKERHUB_REPO        = 'sathiyananth'          // DockerHub username from credentials
        BACKEND_IMAGE         = "${DOCKERHUB_REPO}/medicore-backend"
        FRONTEND_IMAGE        = "${DOCKERHUB_REPO}/medicore-frontend"
        IMAGE_TAG             = "${BUILD_NUMBER}"

        // AWS EC2 Target Server Configuration
        EC2_CREDENTIALS_ID    = 'ec2-ssh-mum'           // Jenkins SSH key credentials ID
        EC2_USER              = 'ubuntu'                // EC2 SSH User
        EC2_HOST              = credentials('ec2-host') // Jenkins Secret Text credential containing your EC2 IP
        DEPLOY_PATH           = '/home/ubuntu/medicore-hms'
    }

    stages {
        stage('Checkout Source') {
            steps {
                echo 'Checking out latest source code...'
                checkout scm
            }
        }

        stage('Code Validation & Lint') {
            steps {
                echo 'Validating TypeScript and compiling codebase...'
                sh '''
                    npm ci --legacy-peer-deps
                    npm run lint
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                echo "Building Docker images with tag: ${IMAGE_TAG} and latest..."
                sh """
                    docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} -t ${BACKEND_IMAGE}:latest -f backend/Dockerfile .
                    docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} -t ${FRONTEND_IMAGE}:latest -f frontend/Dockerfile .
                """
            }
        }

        stage('Push to DockerHub') {
            steps {
                echo 'Authenticating and pushing images to DockerHub...'
                withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDENTIALS}", usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh """
                        echo "\$DH_PASS" | docker login -u "\$DH_USER" --password-stdin
                        docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                        docker push ${BACKEND_IMAGE}:latest
                        docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                        docker push ${FRONTEND_IMAGE}:latest
                        docker logout
                    """
                }
            }
        }

        stage('Deploy to AWS EC2') {
            steps {
                echo "Deploying to AWS EC2 instance (${EC2_HOST})..."
                sshagent([ "${EC2_CREDENTIALS_ID}" ]) {
                    sh """
                        # 1. Prepare target deployment directory on EC2
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} "mkdir -p ${DEPLOY_PATH}"

                        # 2. Copy production docker-compose and deployment configs to EC2
                        scp -o StrictHostKeyChecking=no docker-compose.prod.yml ${EC2_USER}@${EC2_HOST}:${DEPLOY_PATH}/docker-compose.yml

                        # 3. Pull latest images and restart containers on EC2
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            cd ${DEPLOY_PATH}
                            export DOCKER_IMAGE_BACKEND="${BACKEND_IMAGE}:${IMAGE_TAG}"
                            export DOCKER_IMAGE_FRONTEND="${FRONTEND_IMAGE}:${IMAGE_TAG}"
                            docker compose pull
                            docker compose up -d --remove-orphans
                            docker image prune -f
                        '
                    """
                }
            }
        }

        stage('Verify Deployment Health') {
            steps {
                echo 'Running post-deployment health check on EC2...'
                sh """
                    sleep 10
                    curl --fail --retry 5 --retry-delay 3 http://${EC2_HOST}/api/health || exit 1
                """
            }
        }
    }

    post {
        always {
            echo 'Cleaning up local workspace images...'
            sh """
                docker rmi ${BACKEND_IMAGE}:${IMAGE_TAG} ${BACKEND_IMAGE}:latest ${FRONTEND_IMAGE}:${IMAGE_TAG} ${FRONTEND_IMAGE}:latest || true
            """
        }
        success {
            echo "🎉 MediCore HMS successfully deployed to AWS EC2 (http://${EC2_HOST})!"
        }
        failure {
            echo '❌ Pipeline failed! Please check logs for details.'
        }
    }
}

