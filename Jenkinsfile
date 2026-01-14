pipeline {
  agent any

  tools {
    nodejs 'node20'
  }

  environment {
    IMAGE_NAME = "raveenkumarofficial2025/github-profile-summarizer"
    IMAGE_TAG  = "v${BUILD_NUMBER}"
    MAX_REPOS  = "50"
    CONTAINER_NAME = "github-profile-summarizer"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build (Node)') {
      steps {
        sh '''
          npm install
          npm run build
        '''
      }
    }

    stage('Docker Build') {
      steps {
        withCredentials([
          string(credentialsId: 'github-token-secret', variable: 'GITHUB_TOKEN_VALUE')
        ]) {
          sh '''
            docker build \
              --build-arg VITE_GITHUB_TOKEN=$GITHUB_TOKEN_VALUE \
              --build-arg VITE_MAX_REPOS=$MAX_REPOS \
              -t $IMAGE_NAME:$IMAGE_TAG .
          '''
        }
      }
    }

    stage('Docker Login & Push') {
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'DockerHub',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
          )
        ]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push $IMAGE_NAME:$IMAGE_TAG
            docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest
            docker push $IMAGE_NAME:latest
          '''
        }
      }
    }

    stage('Deploy Image') {
      steps {
        sh '''
          docker rm -f $CONTAINER_NAME || true
          docker run -d \
            --name $CONTAINER_NAME \
            -p 8081:80 \
            $IMAGE_NAME:$IMAGE_TAG
        '''
      }
    }
  }

  post {
    success {
      echo "✅ Deployed $IMAGE_NAME:$IMAGE_TAG on port 8081"
    }
    failure {
      echo "❌ Pipeline failed"
    }
  }
}
