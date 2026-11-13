pipeline {
    agent any

    environment {
        // DockerHub Configuration
        DOCKERHUB_CREDENTIALS = 'dockerhub-credentials' // Jenkins Credentials ID (Username with password)
        DOCKERHUB_REPO        = 'sathiyananth'          // DockerHub username
        BACKEND_IMAGE         = "${DOCKERHUB_REPO}/medicore-backend"
        FRONTEND_IMAGE        = "${DOCKERHUB_REPO}/medicore-frontend"
        IMAGE_TAG             = "${BUILD_NUMBER}"

        // AWS EC2 Target Server Configuration
        EC2_CREDENTIALS_ID    = 'ec2-ssh-mum'           // Jenkins SSH key Credentials ID
        EC2_USER              = 'ubuntu'                // EC2 SSH User
        EC2_HOST              = credentials('ec2-host') // Jenkins Secret Text containing your EC2 IP
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
                bat '''
                    call npm ci --legacy-peer-deps
                    call npm run lint
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                echo "Building Docker images with tag: ${IMAGE_TAG} and latest..."
                bat """
                    docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} -t ${BACKEND_IMAGE}:latest -f backend/Dockerfile .
                    if errorlevel 1 exit /b 1
                    docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} -t ${FRONTEND_IMAGE}:latest -f frontend/Dockerfile .
                    if errorlevel 1 exit /b 1
                """
            }
        }

        stage('Push to DockerHub') {
            steps {
                echo 'Authenticating and pushing images to DockerHub...'
                withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDENTIALS}", usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    bat """
                        echo %DH_PASS%| docker login -u %DH_USER% --password-stdin
                        if errorlevel 1 exit /b 1
                        docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                        if errorlevel 1 exit /b 1
                        docker push ${BACKEND_IMAGE}:latest
                        if errorlevel 1 exit /b 1
                        docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                        if errorlevel 1 exit /b 1
                        docker push ${FRONTEND_IMAGE}:latest
                        if errorlevel 1 exit /b 1
                    """
                }
            }
        }

        stage('Deploy to AWS EC2') {
            steps {
                echo "Deploying to AWS EC2 instance..."
                withCredentials([
                    sshUserPrivateKey(credentialsId: "${EC2_CREDENTIALS_ID}", keyFileVariable: 'SSH_KEY_FILE'),
                    usernamePassword(credentialsId: "${DOCKERHUB_CREDENTIALS}", usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')
                ]) {
                    bat """
                        rem 0. Fix Windows permissions on SSH key file (strip BUILTIN\\Users)
                        powershell -Command "\$acl = Get-Acl '%SSH_KEY_FILE%'; \$acl.SetAccessRuleProtection(\$true, \$false); \$rule = New-Object System.Security.AccessControl.FileSystemAccessRule([System.Security.Principal.WindowsIdentity]::GetCurrent().Name, 'FullControl', 'Allow'); \$acl.SetAccessRule(\$rule); Set-Acl '%SSH_KEY_FILE%' \$acl"
                        icacls "%SSH_KEY_FILE%" /remove:g "BUILTIN\\Users" >nul 2>&1
                        icacls "%SSH_KEY_FILE%" /remove:g "Users" >nul 2>&1

                        rem 1. Prepare target deployment directory on EC2
                        ssh -i "%SSH_KEY_FILE%" -o StrictHostKeyChecking=no %EC2_USER%@%EC2_HOST% "mkdir -p %DEPLOY_PATH%"

                        rem 2. Copy production docker-compose file to EC2
                        scp -i "%SSH_KEY_FILE%" -o StrictHostKeyChecking=no docker-compose.prod.yml %EC2_USER%@%EC2_HOST%:%DEPLOY_PATH%/docker-compose.yml

                        rem 3. Authenticate EC2 to DockerHub, pull images, and restart containers
                        ssh -i "%SSH_KEY_FILE%" -o StrictHostKeyChecking=no %EC2_USER%@%EC2_HOST% "echo %DH_PASS% | docker login -u %DH_USER% --password-stdin && cd %DEPLOY_PATH% && export DOCKER_IMAGE_BACKEND=${BACKEND_IMAGE}:${IMAGE_TAG} && export DOCKER_IMAGE_FRONTEND=${FRONTEND_IMAGE}:${IMAGE_TAG} && docker compose pull && docker compose up -d --remove-orphans && docker image prune -f && docker logout"
                    """
                }
            }
        }

        stage('Verify Deployment Health') {
            steps {
                echo "Running post-deployment health check on EC2..."
                bat """
                    curl.exe --fail --retry 5 --retry-delay 3 http://%EC2_HOST%/api/health
                """
            }
        }
    }

    post {
        always {
            echo 'Cleaning up local workspace images...'
            bat """
                docker logout || exit 0
                docker rmi ${BACKEND_IMAGE}:${IMAGE_TAG} ${BACKEND_IMAGE}:latest ${FRONTEND_IMAGE}:${IMAGE_TAG} ${FRONTEND_IMAGE}:latest || exit 0
            """
        }
        success {
            echo "🎉 MediCore HMS successfully deployed to AWS EC2!"
        }
        failure {
            echo '❌ Pipeline failed! Please check logs for details.'
        }
    }
}
