pipeline {
  agent any

  options { timestamps(); ansiColor('xterm'); buildDiscarder(logRotator(numToKeepStr: '20')) }

  environment {
    // Make sure Jenkins can find docker on macOS (Docker Desktop paths)
    PATH          = "/opt/homebrew/bin:/usr/local/bin:${env.PATH}"

    REGISTRY       = 'docker.io'
    FRONTEND_IMAGE = 'tanmayranaware/applens-frontend'
    BACKEND_IMAGE  = 'tanmayranaware/applens-backend'
    EC2_HOST       = 'ubuntu@ec2-3-21-127-72.us-east-2.compute.amazonaws.com'
    TAG            = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Enable Buildx (once per agent)') {
      steps {
        sh '''
          # Ensure a working docker context (Desktop uses "desktop-linux"; fallback to default)
          docker context use default >/dev/null 2>&1 || docker context use desktop-linux >/dev/null 2>&1 || true

          # Create & bootstrap buildx builder (idempotent)
          docker buildx create --use || true
          docker buildx inspect --bootstrap
        '''
      }
    }

    // Build + push AMD64 images so they run on t3 EC2
    stage('Build & Push Images (amd64)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                          usernameVariable: 'DOCKERHUB_USER',
                                          passwordVariable: 'DOCKERHUB_TOKEN')]) {
          sh '''
            echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_USER" --password-stdin docker.io

            docker buildx build --platform linux/amd64 \
              -t ${FRONTEND_IMAGE}:${TAG} -t ${FRONTEND_IMAGE}:latest \
              -f frontend/Dockerfile frontend --push

            docker buildx build --platform linux/amd64 \
              -t ${BACKEND_IMAGE}:${TAG} -t ${BACKEND_IMAGE}:latest \
              -f backend/Dockerfile backend --push

            docker logout docker.io || true
          '''
        }
      }
    }

    stage('Deploy to EC2 (main only)') {
      when { branch 'main' }
      steps {
        sshagent(credentials: ['ec2-ssh']) {
          sh """
            ssh -o StrictHostKeyChecking=no ${EC2_HOST} '
              set -e
              cd /srv/app

              # Update deploy TAG in env file
              if grep -q "^TAG=" .env.prod; then
                sed -i "s/^TAG=.*/TAG=${TAG}/" .env.prod
              else
                echo "TAG=${TAG}" >> .env.prod
              fi

              # Pull new images and restart
              docker compose -f docker-compose.prod.yml --env-file .env.prod pull
              docker compose -f docker-compose.prod.yml --env-file .env.prod up -d
              docker compose -f docker-compose.prod.yml --env-file .env.prod ps
            '
          """
        }
      }
    }
  }

  post {
    success { echo "Deployed ${TAG} to EC2" }
    failure { echo "Pipeline FAILED" }
  }
}
