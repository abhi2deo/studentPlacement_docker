pipeline {
    agent any
    environment {
        IMAGE = "YOUR_DOCKERHUB_USERNAME/placement-portal:${BUILD_NUMBER}"
        DEPLOYMENT = "placement-portal"
        CONTAINER = "web"
    }
    stages {
        stage('Clone Code') {
            steps { checkout scm }
        }
        stage('Build Docker Image') {
            steps { sh 'docker build -t $IMAGE .' }
        }
        stage('Test Application') {
            steps {
                sh '''
                  set -e
                  docker run -d --rm --name app-test -p 18080:80 $IMAGE
                  trap 'docker stop app-test >/dev/null 2>&1 || true' EXIT
                  for i in $(seq 1 15); do
                    if curl -fsS http://127.0.0.1:18080/ >/dev/null; then exit 0; fi
                    sleep 2
                  done
                  echo "Application did not respond"
                  exit 1
                '''
            }
        }
        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                      echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                      docker push "$IMAGE"
                      docker logout
                    '''
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                      export KUBECONFIG="$KUBECONFIG_FILE"
                      kubectl apply -f deployment.yaml
                      kubectl set image deployment/$DEPLOYMENT web=$IMAGE
                      if ! kubectl rollout status deployment/$DEPLOYMENT --timeout=120s; then
                        echo "Rollout failed; restoring previous stable revision"
                        kubectl rollout undo deployment/$DEPLOYMENT
                        kubectl rollout status deployment/$DEPLOYMENT --timeout=120s
                        exit 1
                      fi
                      kubectl get pods -l app=$DEPLOYMENT
                      kubectl get service placement-portal-service
                    '''
                }
            }
        }
    }
    post {
        always { echo 'Pipeline finished. Check stage logs for the result.' }
    }
}
