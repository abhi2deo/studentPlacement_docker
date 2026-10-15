parameters {
    booleanParam(
        name: 'SIMULATE_FAILURE',
        defaultValue: false,
        description: 'Deploy a faulty image to test automatic rollback'
    )
}

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
        stage('Deploy and Verify') {
    steps {
        script {
            def deployImage = params.SIMULATE_FAILURE
                ? 'YOUR_DOCKERHUB_USERNAME/placement-portal:broken-demo'
                : env.IMAGE

            withCredentials([
                file(
                    credentialsId: 'kubeconfig',
                    variable: 'KUBECONFIG_FILE'
                )
            ]) {
                def result = bat(
                    returnStatus: true,
                    script: """
                        @echo off
                        set KUBECONFIG=%KUBECONFIG_FILE%

                        kubectl set image deployment/placement-portal web=${deployImage}

                        if errorlevel 1 exit /b 1

                        kubectl rollout status deployment/placement-portal --timeout=90s

                        if errorlevel 1 (
                            echo FAILED ROLLOUT DETECTED
                            echo Rolling back to the previous stable version...

                            kubectl rollout undo deployment/placement-portal
                            if errorlevel 1 exit /b 1

                            kubectl rollout status deployment/placement-portal --timeout=90s
                            if errorlevel 1 exit /b 1

                            echo ROLLBACK SUCCESSFUL
                            exit /b 1
                        )

                        echo DEPLOYMENT SUCCESSFUL
                    """
                )

                if (result != 0 && !params.SIMULATE_FAILURE) {
                    error('Deployment failed.')
                }
                if (result != 0) {
                    echo 'Failure was detected. Check the rollback logs and verify the restored application.'
                }
            }
        }
    }

    post {
        always { echo 'Pipeline finished. Check stage logs for the result.' }
    }
}
