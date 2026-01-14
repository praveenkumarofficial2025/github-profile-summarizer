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
          // Step 1: Clean up any old containers
          sh '''
            docker rm -f ${CONTAINER_NAME_NEW} ${CONTAINER_NAME_OLD} || true
          '''
          
          // Step 2: Check if current container exists and get its port
          def currentContainerExists = sh(
            script: '''
              if docker ps -a --filter "name=^/${CONTAINER_NAME}$" --format "{{.Names}}" | grep -q "${CONTAINER_NAME}"; then
                echo "true"
              else
                echo "false"
              fi
            ''',
            returnStdout: true
          ).trim() == 'true'
          
          // Step 3: Determine ports
          def currentPort = 8081
          def newPort = 8082
          
          if (currentContainerExists) {
            // Get the port from the running container
            def portInfo = sh(
              script: """
                docker port ${CONTAINER_NAME} 2>/dev/null | head -1 | cut -d: -f2 || echo "8081"
              """,
              returnStdout: true
            ).trim()
            
            if (portInfo.isInteger()) {
              currentPort = portInfo.toInteger()
              newPort = currentPort == 8081 ? 8082 : 8081
            }
          }
          
          echo "Current container exists: ${currentContainerExists}"
          echo "Current port: ${currentPort}"
          echo "New port: ${newPort}"
          
          // Step 4: Deploy new version on different port
          sh """
            # Deploy new container
            docker run -d \\
              --name ${CONTAINER_NAME_NEW} \\
              -p ${newPort}:80 \\
              --health-cmd "curl --fail http://localhost:80 || exit 1" \\
              --health-interval 10s \\
              --health-timeout 5s \\
              --health-retries 5 \\
              --health-start-period 30s \\
              ${IMAGE_NAME}:${IMAGE_TAG}
            
            echo "✅ New version deployed on port ${newPort}"
          """
          
          // Step 5: Wait for health check with timeout
          timeout(time: 3, unit: 'MINUTES') {
            waitUntil {
              try {
                def health = sh(
                  script: """
                    docker inspect --format='{{.State.Health.Status}}' ${CONTAINER_NAME_NEW} 2>/dev/null || echo "starting"
                  """,
                  returnStdout: true
                ).trim()
                
                echo "Health status of new container: ${health}"
                
                if (health == 'healthy') {
                  return true
                } else if (health == 'unhealthy') {
                  error "New container is unhealthy. Deployment failed."
                }
                return false
              } catch (Exception e) {
                echo "Waiting for container to start..."
                return false
              }
            }
          }
          
          echo "✅ New container is healthy"
          
          // Step 6: Traffic switching strategy
          if (currentContainerExists) {
            echo "🔄 Switching traffic from old to new version..."
            
            // Option A: Using reverse proxy (recommended for production)
            // Option B: Port swapping with minimal downtime
            
            // For minimal downtime port swap:
            sh """
              # Stop old container
              docker stop ${CONTAINER_NAME}
              
              # Rename containers
              docker rename ${CONTAINER_NAME} ${CONTAINER_NAME_OLD}
              docker rename ${CONTAINER_NAME_NEW} ${CONTAINER_NAME}
              
              # Remove old container
              docker rm -f ${CONTAINER_NAME_OLD} || true
              
              # Update port mapping for the renamed container
              docker stop ${CONTAINER_NAME}
              docker rm ${CONTAINER_NAME}
              
              # Run on the original port
              docker run -d \\
                --name ${CONTAINER_NAME} \\
                -p ${currentPort}:80 \\
                --health-cmd "curl --fail http://localhost:80 || exit 1" \\
                --health-interval 10s \\
                --health-retries 3 \\
                ${IMAGE_NAME}:${IMAGE_TAG}
            """
            
            // Verify new container on original port is healthy
            timeout(time: 1, unit: 'MINUTES') {
              waitUntil {
                def health = sh(
                  script: """
                    docker inspect --format='{{.State.Health.Status}}' ${CONTAINER_NAME} 2>/dev/null || echo "starting"
                  """,
                  returnStdout: true
                ).trim() == 'healthy'
                return health
              }
            }
            
            echo "✅ Traffic switched successfully"
            
          } else {
            // First deployment
            sh """
              docker rename ${CONTAINER_NAME_NEW} ${CONTAINER_NAME}
            """
            echo "✅ First deployment completed"
          }
          
          // Step 7: Cleanup
          sh '''
            docker rm -f ${CONTAINER_NAME_NEW} ${CONTAINER_NAME_OLD} 2>/dev/null || true
            docker image prune -f
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅ Successfully deployed ${env.IMAGE_NAME}:${env.IMAGE_TAG}"
      sh '''
        echo "=== Deployment Summary ==="
        docker ps --filter "name=github-profile-summarizer" --format "table {{.Names}}\t{{.Ports}}\t{{.Status}}\t{{.Image}}"
        echo "=========================="
      '''
    }
    failure {
      echo "❌ Deployment failed - rolling back"
      script {
        // Rollback: Ensure original container is running
        sh '''
          # Check if original container exists
          if docker ps -a --filter "name=^/github-profile-summarizer$" --format "{{.Names}}" | grep -q "github-profile-summarizer"; then
            echo "Starting original container..."
            docker start github-profile-summarizer || true
          fi
          
          # Clean up failed new containers
          docker rm -f github-profile-summarizer-new github-profile-summarizer-old 2>/dev/null || true
          
          echo "Rollback completed. Original version restored."
        '''
      }
    }
  }
}
