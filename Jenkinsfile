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

    stage('Push to Docker Hub') {
        steps {
            withCredentials([
                usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
            sh '''
            IMAGE="$DOCKER_USER/holiday-events"

            docker tag holiday-events:candidate "$IMAGE:${BUILD_NUMBER}"
            docker tag holiday-events:candidate "$IMAGE:latest"

            echo "$DOCKER_TOKEN" | docker login -u "$DOCKER_USER" --password-stdin
            docker push "$IMAGE:${BUILD_NUMBER}"
            docker push "$IMAGE:latest"
            docker logout

            docker image rm -f \
                holiday-events:candidate \
                "$IMAGE:${BUILD_NUMBER}" \
                "$IMAGE:latest"
            '''
        }
      }
    }

    stage('Deploy with Ansible') {
        steps {
            sshagent(credentials: ['ubuntu-vm-ssh']) {
            withCredentials([
                string(
                credentialsId: 'ubuntu-sudo',
                variable: 'ANSIBLE_BECOME_PASSWORD'
                )
            ]) {
                sh '''
                umask 077
                trap 'rm -f .ansible-secrets.json' EXIT

                python3 -c '
        import json, os
        print(json.dumps({
        "ansible_become_password": os.environ["ANSIBLE_BECOME_PASSWORD"]
        }))
        ' > .ansible-secrets.json

                ansible-playbook \
                    -i ansible/inventory.ini \
                    ansible/deploy.yml \
                    -e @.ansible-secrets.json \
                    -e "app_image=yanivking06/holiday-events:${BUILD_NUMBER}"
                '''
            }
            }
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