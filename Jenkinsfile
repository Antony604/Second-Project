pipeline {
  agent any
  environment {
    IMAGE     = 'antonypeter02/second-project'
    TAG       = "${env.BUILD_NUMBER}"
    CONTAINER = 'second-project'
    HOST_PORT = '80'
    APP_PORT  = '8080'
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Build image') {
      steps { sh 'docker build -t $IMAGE:$TAG .' }
    }
    stage('Push to registry') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                         usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh 'echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin'
          sh 'docker push $IMAGE:$TAG'
        }
      }
    }
    stage('Deploy') {
      steps {
        sh '''
          docker inspect -f '{{.Config.Image}}' $CONTAINER > prev_image.txt 2>/dev/null || true
          docker rm -f $CONTAINER || true
          docker run -d --name $CONTAINER --restart unless-stopped \
            --memory=128m --cpus=0.25 \
            -p $HOST_PORT:$APP_PORT $IMAGE:$TAG
        '''
      }
    }
    stage('Health check') {
      steps {
        sh '''
          for i in $(seq 1 12); do
            STATUS=$(docker inspect -f '{{.State.Health.Status}}' $CONTAINER)
            echo "health: $STATUS"
            if [ "$STATUS" = "healthy" ] && curl -fs http://localhost:$HOST_PORT/ > /dev/null; then
              exit 0
            fi
            sleep 5
          done
          exit 1
        '''
      }
    }
  }
  post {
    success { echo "Deployed $IMAGE:$TAG" }
    failure {
      sh '''
        PREV=$(cat prev_image.txt 2>/dev/null || true)
        CURRENT=$(docker inspect -f '{{.Config.Image}}' $CONTAINER 2>/dev/null || true)
        if [ -n "$PREV" ] && [ "$CURRENT" = "$IMAGE:$TAG" ]; then
          echo "Rolling back to $PREV"
          docker rm -f $CONTAINER
          docker run -d --name $CONTAINER --restart unless-stopped \
            --memory=128m --cpus=0.25 -p $HOST_PORT:$APP_PORT $PREV
        fi
      '''
    }
    always { sh 'docker logout || true' }
  }
}
