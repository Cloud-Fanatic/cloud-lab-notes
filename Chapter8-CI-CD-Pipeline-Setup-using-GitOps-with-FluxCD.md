# Chapter 8 — CI/CD Pipeline Setup — GitOps with Git Actions and FluxCD

## Step 0 - Set up AWS ECR (Elastic Container Registry)

Create a new Container Registry in AWS ECR (Private Repo):

![ECR Set up in AWS](/images/12.jpg)


## Step 1 — Set up GitHub Repository & GitOps Folder Structure

I had already created the repo 'cloud-lab'.

Structured the repo for FluxCD + Kubernetes manifests:

```
cloud-lab/
├─ .github/workflows/ci-cd-pipeline.yml  # GitHub Action workflow
├─ k8s/
│  ├─ deployment.yaml
│  ├─ image-repo.yaml
│  ├─ image-policy.yaml
│  ├─ ImageUpdateAutomation.yaml
│  └─ kustomization.yaml
├─ VERSION.txt
```

- Ensured deployment.yaml referenced your ECR image.

- Added # kustomize: name=image and # kustomize: tag=latest comments to enable FluxCD image updates.

## Step 2 — GitHub Actions CI/CD Workflow

- Workflow trigger: push to VERSION.txt file.

- Steps implemented:

1.  Checkout repository (actions/checkout@v3).

2.  Set a timestamp as IMAGE_TAG for unique builds.

3.  Login to AWS ECR (aws-actions/amazon-ecr-login@v2).

4.  Build Docker image and push both timestamp tag and latest tag.

5.  Update k8s/image-repo.yaml with new image information.

6.  Update the ci-cd-pipelines.yaml manifest file, using our git secret variables:
   
![ECR variable name](/images/13-1.jpg)

<img src="/images/13-2.jpg" alt="ECR variable name in manifest file" width="600" height="400">

## Step 3 — FluxCD GitOps Setup

1.  ImageRepository

- Watches your ECR repo for updates.
- References your ECR registry and aws-ecr-credentials secret.

2. ImagePolicy

- Picks the latest tag in ECR automatically (or uses semantic versions if configured).

4.  ImageUpdateAutomation

- Watches ImagePolicy for new tags.
- Automatically patches Deployment manifest with the latest image tag.
  
5.  Kustomization

- Applies your k8s/ folder manifests to the K3s cluster.
- Prunes old resources to keep the cluster clean.

- Key fix: Always push :latest tag so K3s Deployment can pull it successfully.

- Outcome: GitHub Action builds an image every time VERSION.txt is updated.

## Step 4 — Kubernetes Deployment Setup

- deployment.yaml configured:
- Container points to ECR image.

- Pull secret references aws-ecr-credentials.
-- # kustomize: name=image and # kustomize: tag=latest added for Flux patching.

Secret creation (on EC2/K3s) to allow image pulls from ECR:

```
aws ecr get-login-password --region us-east-1 | \
kubectl create secret docker-registry aws-ecr-credentials \
--docker-server=<ECR_URI> --docker-username=AWS --docker-password-stdin --namespace default
```
- Ensured old pods were deleted so new pods pull the updated image.

## Step 5 — Troubleshooting / Key Learning Points

1.  Error “not found”: K3s couldn’t pull :latest because the image was tagged with a timestamp. Fix: GitHub Action now pushes latest in addition to timestamp.

2.  ImagePullBackOff: Pod couldn’t authenticate to ECR. Fix: Created aws-ecr-credentials secret.

3.  Deployment YAML must reference the correct ECR image — otherwise pods still pull public images like nginx:alpine.

4.  FluxCD automation only works if Deployment contains # kustomize setters and matches ImageRepository.

## Step 6 — Version visibility

- Embedded version info in Docker image or tag based on VERSION.txt.

- Checked via `kubectl`:
```
# Check image tag
kubectl get pod <pod-name> -n default -o jsonpath='{.spec.containers[0].image}'

# If embedded in container:
kubectl exec -it <pod-name> -- cat /app/VERSION.txt
```

## Step 7 — Final result

Push to VERSION.txt → GitHub Action builds & pushes image → FluxCD detects → Deployment updated → K3s pod pulls new ECR image automatically.

In the below example, we can confirm that on k3s when a new container is created, it now pulls our custom Container from our AWS ECR, always the latest version, and this happens everytime a git commit/ change is made to the VERSION.txt file, just as a use-case example for this Cloud Lab demonstration:

![ECR custom image created in k3s](/images/14.jpg)

✅ Fully automated CI/CD from GitHub → ECR → K3s using FluxCD GitOps.
