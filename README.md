# Q1-Student-Placement-Portal

## Files
- `index.html`: application interface
- `Dockerfile`: packages the page in Nginx
- `deployment.yaml`: Kubernetes Deployment and NodePort Service
- `Jenkinsfile`: CI/CD pipeline with build, smoke test, push, deploy and rollout rollback

## Configure before running
1. Replace `YOUR_DOCKERHUB_USERNAME` in `deployment.yaml` and `Jenkinsfile`.
2. In Jenkins, create a username/password credential with ID `dockerhub-creds`.
3. Create a Jenkins **Secret file** credential with ID `kubeconfig`, containing a kubeconfig that can access the target cluster.
4. Use a Jenkins agent with Docker CLI/daemon, kubectl, curl and access to Docker Hub and the Kubernetes API.
5. Create a Jenkins Pipeline job connected to this Git repository. Commit the `Jenkinsfile` at the repository root.
6. Ensure Docker Desktop Kubernetes is enabled and the Jenkins agent's kubeconfig targets the correct cluster.

## Local test
```bash
docker build -t placement-portal:local .
docker run --rm -p 8080:80 placement-portal:local
```
Open http://localhost:8080

## Push image manually
```bash
docker login
docker build -t YOUR_DOCKERHUB_USERNAME/placement-portal:latest .
docker push YOUR_DOCKERHUB_USERNAME/placement-portal:latest
```

## Deploy and verify manually
Replace the image placeholder in `deployment.yaml`, then run:
```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/placement-portal
kubectl get pods -o wide
kubectl get service placement-portal-service
```
Open `http://localhost:30081` when using Docker Desktop Kubernetes.

## Demonstrate rollback
The Jenkinsfile automatically runs `kubectl rollout undo` if rollout status fails. To create a controlled failure for a lab demo, first ensure a successful stable deployment exists, then use:
```bash
kubectl set image deployment/placement-portal web=YOUR_DOCKERHUB_USERNAME/placement-portal:missing-tag
kubectl rollout status deployment/placement-portal --timeout=60s
kubectl rollout undo deployment/placement-portal
kubectl rollout status deployment/placement-portal
```
Use the Jenkins pipeline to demonstrate automated rollback; the manual commands are for observing the Kubernetes behavior. The pipeline rollback logic triggers on a failed rollout of an image it deploys.
