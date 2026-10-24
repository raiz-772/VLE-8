# VLE8 Blue-Green Deployment

A starter project demonstrating a blue-green deployment workflow using **Jenkins, Docker, and Kubernetes**.

## Files

- `Dockerfile` — builds an NGINX image containing the demo page.
- `index.html` — sample web page.
- `deployment-blue.yaml` — blue Kubernetes Deployment.
- `deployment-green.yaml` — green Kubernetes Deployment.
- `service.yaml` — Service that routes traffic to the selected version.
- `Jenkinsfile` — builds and pushes the image, deploys both versions, and switches Service traffic.

## Before running

1. Push this folder to your GitHub repository.
2. Update `IMAGE_NAME` in `Jenkinsfile` and both deployment YAML files if your Docker Hub repository is different.
3. In Jenkins, install/configure the required Docker and Kubernetes command-line tools on the agent.
4. Add Jenkins credentials:
   - `dockerhub-credentials`: **Username with password** credentials for Docker Hub.
   - `kubeconfig`: **Secret file** containing a kubeconfig that can access your Kubernetes cluster.
5. Ensure the Jenkins agent can run Docker and that the Kubernetes cluster can pull your Docker Hub image.

## Run locally

Build the container:

```bash
docker build -t vle8-blue-green:local .
docker run --rm -p 8080:80 vle8-blue-green:local
```

Open `http://localhost:8080`.

## Deploy manually

Update the image name in both deployment manifests, then run:

```bash
kubectl apply -f deployment-blue.yaml
kubectl apply -f deployment-green.yaml
kubectl apply -f service.yaml
kubectl get deployments,pods,svc
```

The Service initially routes traffic to **blue**. Switch traffic to green:

```bash
kubectl patch service vle8-service --type merge -p '{"spec":{"selector":{"app":"vle8-demo","version":"green"}}}'
```

Switch back to blue:

```bash
kubectl patch service vle8-service --type merge -p '{"spec":{"selector":{"app":"vle8-demo","version":"blue"}}}'
```

## Notes

- The Jenkins pipeline uses the `DEPLOY_VERSION` parameter to choose which version receives traffic.
- This is a lab/demo template. Configure credentials securely in Jenkins; do not commit passwords, tokens, or kubeconfig files to GitHub.
- For a production setup, add health checks, resource requests/limits, image scanning, approval gates, and a rollback strategy.
