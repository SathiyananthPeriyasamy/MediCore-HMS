pipeline {
    agent any

    environment {
        // DockerHub Configuration
        DOCKERHUB_CREDENTIALS = 'dockerhub-credentials' // Jenkins Credentials ID (Username with password)
        DOCKERHUB_REPO        = 'sathiyananth'          // DockerHub username
        BACKEND_IMAGE         = "${DOCKERHUB_REPO}/medicore-backend"
        FRONTEND_IMAGE        = "${DOCKERHUB_REPO}/medicore-frontend"
        IMAGE_TAG             = "${BUILD_NUMBER}"

        // Physical Target Server Configuration (Company On-Premise Server)
        SERVER_HOST           = credentials('serverip') // Injected dynamically from Jenkins secret text 'serverip'
        SERVER_USER           = 'cubeai'                // Target Linux user on physical server
        SERVER_CREDENTIALS_ID = 'server-cred'           // Matches your SSH private key Credential ID
        DEPLOY_PATH           = "/home/${SERVER_USER}/medicore-hms"
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
                    docker build --network=host -t ${BACKEND_IMAGE}:${IMAGE_TAG} -t ${BACKEND_IMAGE}:latest -f backend/Dockerfile .
                    if errorlevel 1 exit /b 1
                    docker build --network=host -t ${FRONTEND_IMAGE}:${IMAGE_TAG} -t ${FRONTEND_IMAGE}:latest -f frontend/Dockerfile .
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

        stage('Deploy to Physical Server') {
            steps {
                echo "Deploying to Physical Server (${SERVER_HOST})..."
                withCredentials([
                    sshUserPrivateKey(credentialsId: "${SERVER_CREDENTIALS_ID}", keyFileVariable: 'SSH_KEY_FILE'),
                    usernamePassword(credentialsId: "${DOCKERHUB_CREDENTIALS}", usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')
                ]) {
                    bat """
                        rem 0. Fix Windows permissions on SSH key file (strip BUILTIN\\Users)
                        powershell -Command "\$acl = Get-Acl '%SSH_KEY_FILE%'; \$acl.SetAccessRuleProtection(\$true, \$false); \$rule = New-Object System.Security.AccessControl.FileSystemAccessRule([System.Security.Principal.WindowsIdentity]::GetCurrent().Name, 'FullControl', 'Allow'); \$acl.SetAccessRule(\$rule); Set-Acl '%SSH_KEY_FILE%' \$acl"
                        icacls "%SSH_KEY_FILE%" /remove:g "BUILTIN\\Users" >nul 2>&1
                        icacls "%SSH_KEY_FILE%" /remove:g "Users" >nul 2>&1

                        rem 1. Prepare target deployment directory on server
                        ssh -i "%SSH_KEY_FILE%" -o StrictHostKeyChecking=no %SERVER_USER%@%SERVER_HOST% "mkdir -p %DEPLOY_PATH%"

                        rem 2. Copy production docker-compose file to server
                        scp -i "%SSH_KEY_FILE%" -o StrictHostKeyChecking=no docker-compose.prod.yml %SERVER_USER%@%SERVER_HOST%:%DEPLOY_PATH%/docker-compose.yml

                        rem 3. Authenticate server to DockerHub, pull images, and restart containers
                        ssh -i "%SSH_KEY_FILE%" -o StrictHostKeyChecking=no %SERVER_USER%@%SERVER_HOST% "echo %DH_PASS% | docker login -u %DH_USER% --password-stdin && cd %DEPLOY_PATH% && export DOCKER_IMAGE_BACKEND=${BACKEND_IMAGE}:${IMAGE_TAG} && export DOCKER_IMAGE_FRONTEND=${FRONTEND_IMAGE}:${IMAGE_TAG} && docker compose pull && docker compose up -d --remove-orphans && docker image prune -f && docker logout"
                    """
                }
            }
        }

        stage('Verify Deployment Health') {
            steps {
                echo "Running post-deployment health check on server (${SERVER_HOST})..."
                bat """
                    curl.exe --fail --retry 5 --retry-delay 3 http://%SERVER_HOST%/api/health
                """
            }
        }
    }

    post {
        always {
            echo 'Cleaning up local workspace images...'
            bat '''
                docker logout || exit 0
                docker rmi %BACKEND_IMAGE%:%IMAGE_TAG% %BACKEND_IMAGE%:latest %FRONTEND_IMAGE%:%IMAGE_TAG% %FRONTEND_IMAGE%:latest 2>nul || exit 0
            '''
        }
        success {
            echo "🎉 MediCore HMS successfully deployed to Physical Server!"
        }
        failure {
            echo '❌ Pipeline failed! Please check logs for details.'
        }
    }
}
