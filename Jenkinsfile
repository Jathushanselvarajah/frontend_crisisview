pipeline {
    agent {
        label 'agent1'
    }

    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Branch to build')
        string(name: 'FRONTEND_HOST', defaultValue: '192.168.64.9', description: 'UTM frontend VM IP')
        string(name: 'FRONTEND_USER', defaultValue: 'user', description: 'SSH user for frontend VM')
        string(name: 'REMOTE_APP_DIR', defaultValue: '/home/user/crisisview-frontend', description: 'Remote deployment directory')
        string(name: 'FRONTEND_IMAGE', defaultValue: 'jathus/crisisview-frontend', description: 'Docker Hub frontend image')
        string(name: 'NEXT_PUBLIC_API_URL', defaultValue: 'http://192.168.64.8:3001', description: 'Backend API URL exposed to browser')
        string(name: 'DOCKERHUB_CREDENTIALS_ID', defaultValue: 'dockerhub-credentials', description: 'Jenkins Docker Hub credentials ID')
        string(name: 'SSH_CREDENTIALS_ID', defaultValue: 'utm-ssh-key', description: 'Jenkins SSH credentials ID')
        string(name: 'SONAR_HOST_URL', defaultValue: 'http://sonarqube:9000', description: 'SonarQube URL from Jenkins network')
        string(name: 'SONAR_TOKEN_CREDENTIALS_ID', defaultValue: 'sonarqube-token', description: 'Jenkins SonarQube token credentials ID')
    }

    environment {
        FRONTEND_HOST = "${params.FRONTEND_HOST}"
        FRONTEND_USER = "${params.FRONTEND_USER}"
        REMOTE_APP_DIR = "${params.REMOTE_APP_DIR}"
        FRONTEND_IMAGE = "${params.FRONTEND_IMAGE}"
        NEXT_PUBLIC_API_URL = "${params.NEXT_PUBLIC_API_URL}"
        SONAR_HOST_URL = "${params.SONAR_HOST_URL}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }

        stage('Build Application') {
            steps {
                sh 'NEXT_PUBLIC_API_URL="$NEXT_PUBLIC_API_URL" npm run build'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: params.SONAR_TOKEN_CREDENTIALS_ID, variable: 'SONAR_TOKEN')]) {
                    sh '''
                        sonar-scanner \
                          -Dsonar.host.url="$SONAR_HOST_URL" \
                          -Dsonar.token="$SONAR_TOKEN"
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      --build-arg NEXT_PUBLIC_API_URL="$NEXT_PUBLIC_API_URL" \
                      -t "$FRONTEND_IMAGE:$BUILD_NUMBER" \
                      -t "$FRONTEND_IMAGE:latest" \
                      .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: params.DOCKERHUB_CREDENTIALS_ID, usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_TOKEN')]) {
                    sh '''
                        echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_USER" --password-stdin
                        docker push "$FRONTEND_IMAGE:$BUILD_NUMBER"
                        docker push "$FRONTEND_IMAGE:latest"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy To UTM Frontend') {
            steps {
                sshagent(credentials: [params.SSH_CREDENTIALS_ID]) {
                    sh '''
                        ssh -o StrictHostKeyChecking=accept-new "$FRONTEND_USER@$FRONTEND_HOST" "mkdir -p '$REMOTE_APP_DIR'"

                        ssh "$FRONTEND_USER@$FRONTEND_HOST" "cat > '$REMOTE_APP_DIR/docker-compose.yml'" <<EOF
services:
  frontend:
    image: ${FRONTEND_IMAGE}:latest
    container_name: crisisview-frontend
    restart: unless-stopped
    environment:
      NEXT_PUBLIC_API_URL: ${NEXT_PUBLIC_API_URL}
    ports:
      - "80:3000"
EOF

                        ssh "$FRONTEND_USER@$FRONTEND_HOST" "cd '$REMOTE_APP_DIR' && docker compose pull && docker compose up -d"
                    '''
                }
            }
        }

        stage('Verify Staging Frontend') {
            steps {
                sh '''
                    for i in $(seq 1 30); do
                      if curl -fsS "http://$FRONTEND_HOST"; then
                        exit 0
                      fi
                      sleep 2
                    done
                    exit 1
                '''
            }
        }
    }
}
