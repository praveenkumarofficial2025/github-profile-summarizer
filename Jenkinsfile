pipeline {
  agent any

  environment {
    IMAGE_NAME = "theshubhamgour/github-profile-summarizer"
    IMAGE_TAG = "v${env.BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build (Node)') {
      steps {
        sh 'npm install'
        sh 'npm run build'
      }
    }

    stage('Docker Build') {
      steps {
        // Securely retrieve the GitHub Token from Jenkins Credentials
        // REPLACE 'github-token-secret' with your actual Jenkins Credential ID
        withCredentials([string(credentialsId: 'github-gpg', variable: 'GITHUB_TOKEN_VALUE')]) {
          
          // Pass the token to the Docker build command using --build-arg
          sh "docker build \
            --build-arg VITE_GITHUB_TOKEN=${GITHUB_TOKEN_VALUE} \
            --build-arg VITE_MAX_REPOS=50 \
            -t ${IMAGE_NAME}:${IMAGE_TAG} ."
        }
      }
    }

    stage('Docker Login & Push') {
      steps {
        // Assuming 'DockerHub' is the ID for your DockerHub Username/Password credential
        withCredentials([usernamePassword(credentialsId: 'DockerHub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
        }
        sh 'docker push $IMAGE_NAME:$IMAGE_TAG'
        sh 'docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest'
        sh 'docker push $IMAGE_NAME:latest'
      }
    }

  stage('Deploy image'){
    steps{
      // This step assumes you are running this on the EC2 instance itself.
      // You may need to adjust the port (8081:80) and add a stop/remove for the old container.
      sh 'docker run -d -p 8081:80 $IMAGE_NAME:$IMAGE_TAG'
    }
   }
  }

  post {
    always {
      echo "✅ Image pushed: $IMAGE_NAME:$IMAGE_TAG"
    }
  }
}
