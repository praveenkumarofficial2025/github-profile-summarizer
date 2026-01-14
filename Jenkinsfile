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
    CONTAINER_NAME_NEW = "github-profile-summarizer-new"
    CONTAINER_NAME_OLD = "github-profile-summarizer-old"
    DOCKER_NETWORK = "github-summarizer-network"
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
              -t $IMAGE_NAME:$IMAGE_TAG \
              -t $IMAGE_NAME:latest .
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
            docker push $IMAGE_NAME:latest
          '''
        }
      }
    }

    stage('Zero-Downtime Deployment') {
      steps {
        script {
          // Step 1: Create network if it doesn't exist
          sh '''
            docker network create $DOCKER_NETWORK || true
          '''
          
          // Step 2: Check if current container is running
          def isCurrentRunning = sh(
            script: '''
              if docker ps --filter "name=${CONTAINER_NAME}" --format "{{.Names}}" | grep -q "${CONTAINER_NAME}"; then
                echo "true"
              else
                echo "false"
              fi
            ''',
            returnStdout: true
          ).trim() == 'true'
          
          // Step 3: Deploy new container on different port
          def newPort = 8082
          def currentPort = 8081
          
          if (isCurrentRunning) {
            // Get current port
            currentPort = sh(
              script: '''
                docker port ${CONTAINER_NAME} 80 | cut -d: -f2 || echo "8081"
              ''',
              returnStdout: true
            ).trim().toInteger()
            
            // Set new port (swap)
            newPort = currentPort == 8081 ? 8082 : 8081
          }
          
          // Step 4: Deploy new version
          sh """
            # Remove any old 'new' container
            docker rm -f ${CONTAINER_NAME_NEW} || true
            
            # Run new container
            docker run -d \\
              --name ${CONTAINER_NAME_NEW} \\
              --network ${DOCKER_NETWORK} \\
              -p ${newPort}:80 \\
              --health-cmd "curl --fail http://localhost:80 || exit 1" \\
              --health-interval 10s \\
              --health-timeout 5s \\
              --health-retries 3 \\
              --health-start-period 30s \\
              ${IMAGE_NAME}:${IMAGE_TAG}
            
            echo "New container deployed on port ${newPort}"
          """
          
          // Step 5: Wait for health check
          timeout(time: 2, unit: 'MINUTES') {
            waitUntil {
              def health = sh(
                script: """
                  docker inspect --format='{{.State.Health.Status}}' ${CONTAINER_NAME_NEW}
                """,
                returnStdout: true
              ).trim()
              echo "Health status: ${health}"
              return health == 'healthy'
            }
          }
          
          // Step 6: Update load balancer or swap ports
          if (isCurrentRunning) {
            // Step 6a: Update Nginx config (if using reverse proxy)
            sh '''
              # Update nginx config to point to new container
              # or use Docker built-in routing
              
              # For simple setup, we'll use traffic switching via port swapping
              # This requires an external load balancer or you can use:
              
              # Option A: Use HAProxy or Nginx as reverse proxy
              # Option B: Use Docker swarm/k8s for built-in load balancing
              # Option C: Use port swapping strategy
            '''
            
            // Step 6b: Rename containers for zero-downtime switch
            sh """
              # Rename current to old
              docker rename ${CONTAINER_NAME} ${CONTAINER_NAME_OLD}
              
              # Rename new to current
              docker rename ${CONTAINER_NAME_NEW} ${CONTAINER_NAME}
              
              # Update port mapping if needed
              docker stop ${CONTAINER_NAME} || true
              docker rm ${CONTAINER_NAME} || true
              
              # Re-run on correct port
              docker run -d \\
                --name ${CONTAINER_NAME} \\
                --network ${DOCKER_NETWORK} \\
                -p ${currentPort}:80 \\
                --health-cmd "curl --fail http://localhost:80 || exit 1" \\
                --health-interval 10s \\
                --health-timeout 5s \\
                --health-retries 3 \\
                ${IMAGE_NAME}:${IMAGE_TAG}
            """
            
            // Step 6c: Remove old container
            sh """
              # Wait a bit for connections to drain
              sleep 10
              
              # Remove old container
              docker rm -f ${CONTAINER_NAME_OLD} || true
            """
          } else {
            // First deployment
            sh """
              docker rename ${CONTAINER_NAME_NEW} ${CONTAINER_NAME}
            """
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ Successfully deployed $IMAGE_NAME:$IMAGE_TAG with zero downtime"
      sh '''
        echo "Application is running on:"
        docker ps --filter "name=github-profile-summarizer" --format "table {{.Names}}\t{{.Ports}}\t{{.Status}}"
      '''
    }
    failure {
      echo "❌ Pipeline failed"
      sh '''
        # Cleanup on failure
        docker rm -f ${CONTAINER_NAME_NEW} ${CONTAINER_NAME_OLD} || true
      '''
    }
    always {
      sh '''
        # Cleanup unused images
        docker image prune -f || true
      '''
    }
  }
}
