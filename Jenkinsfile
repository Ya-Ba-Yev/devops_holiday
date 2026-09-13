pipeline {
  agent any

  options {
    skipDefaultCheckout(true)
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build candidate') {
      steps {
        sh 'docker build --target prod -t holiday-events:candidate .'
      }
    }

    stage('Health check') {
      steps {
        sh '''
          TEST_CONTAINER=holiday-events-healthcheck

          docker rm -f "$TEST_CONTAINER" 2>/dev/null || true
          docker run -d --name "$TEST_CONTAINER" holiday-events:candidate

          for i in $(seq 1 20); do
            if docker exec "$TEST_CONTAINER" node -e "
              fetch('http://127.0.0.1:3000/health')
                .then(async response => {
                  const body = await response.json();
                  process.exit(response.ok && body.status === 'healthy' ? 0 : 1);
                })
                .catch(() => process.exit(1));
            "; then
              docker rm -f "$TEST_CONTAINER"
              exit 0
            fi
            sleep 2
          done

          docker logs "$TEST_CONTAINER"
          docker rm -f "$TEST_CONTAINER"
          exit 1
        '''
      }
    }

    stage('Publish and run') {
      steps {
        sh '''
          docker tag holiday-events:candidate holiday-events:latest
          docker rm -f holiday-events 2>/dev/null || true
          docker run -d --name holiday-events -p 8080:3000 holiday-events:latest
          docker image rm holiday-events:candidate
        '''
      }
    }
  }

  post {
    failure {
      sh '''
        docker rm -f holiday-events-healthcheck 2>/dev/null || true
        docker image rm -f holiday-events:candidate 2>/dev/null || true
      '''
    }
  }
}