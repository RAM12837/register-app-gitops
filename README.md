# Register App: GitOps and CD Repository

This repository contains the Kubernetes desired state and the Jenkins CD pipeline for the register application.

## Responsibility

The CD job receives an image tag from the CI job, updates `deployment.yaml`, commits the desired-state change to Git, and pushes it to the `main` branch. Argo CD watches this repository and synchronizes the change to Amazon EKS.

## Repository Contents

- `deployment.yaml`: Kubernetes Deployment with two replicas and the application image.
- `service.yaml`: LoadBalancer Service exposing port `8080`.
- `Jenkinsfile`: parameterized CD pipeline.

## CD Stages

1. Clean the Jenkins workspace.
2. Check out the `main` branch.
3. Read the `IMAGE_TAG` parameter from CI.
4. Replace the image in `deployment.yaml` with `remson001/register-app-pipeline:<IMAGE_TAG>`.
5. Commit the manifest change.
6. Push the commit to GitHub.
7. Let Argo CD detect and synchronize the Git change to EKS.

## Commands on the Jenkins Agent

The CD job runs these commands from a checkout of this repository on the node labeled `Jenkins-Agent`:

```bash
git checkout main
grep 'image:' deployment.yaml
sed -i "s|image: remson001/register-app-pipeline:.*|image: remson001/register-app-pipeline:${IMAGE_TAG}|g" deployment.yaml
git diff -- deployment.yaml
git add deployment.yaml
git commit -m "Updated image tag" || true
git push origin main
```

`IMAGE_TAG` is passed by the CI job. The `sed` command changes the desired image version, the diff confirms the change, and the push makes the new desired state available to Argo CD. GitHub authentication must come from Jenkins Credentials Manager.

## Commands on the EKS Bootstrap Server

Use these commands after Argo CD synchronizes the GitOps commit:

```bash
kubectl get applications -n argocd
kubectl get deployment register-app-deployment
kubectl get pods -l app=register-app -o wide
kubectl get svc register-app-service
kubectl rollout status deployment/register-app-deployment
kubectl logs -l app=register-app --tail=100
```

These commands verify Argo CD health, the Deployment rollout, pod placement, the AWS LoadBalancer address, and application output. The application is reachable through the external Service address on port `8080`.

## Argo CD Deployment Setup Used

Run these steps on the EKS bootstrap server after the cluster has been created and the server IAM role has been attached:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd
cd /tmp
curl --silent --location -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/argocd
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"LoadBalancer"}}'
kubectl get svc argocd-server -n argocd
```

The namespace and manifest install Argo CD on EKS. The CLI is used to communicate with the Argo CD API server, and the service patch provisions an AWS LoadBalancer for external access.

Retrieve the initial password, log in, and register the EKS cluster:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o yaml
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode; echo
argocd login <ARGOCD_LOADBALANCER_HOSTNAME> --username admin
kubectl config get-contexts
argocd cluster add <EKS_CONTEXT_NAME> --name <ARGOCD_CLUSTER_ALIAS>
argocd cluster list
```

The context returned by `kubectl config get-contexts` is the value passed to `argocd cluster add`. The command creates the service account and role binding Argo CD needs to manage the EKS cluster. Change the initial Argo CD password in the UI after the first login.

## Create the Argo CD Application

Register this GitOps repository in **Settings > Repositories**, then create an Application with:

- Repository: `register-app-gitops`.
- Path: `./`.
- Destination: the registered EKS cluster.
- Namespace: `default`.

The equivalent CLI commands are:

```bash
argocd repo add https://github.com/RAM12837/register-app-gitops.git --username <GITHUB_USERNAME> --password '<GITHUB_PAT>'
argocd app create register-app --repo https://github.com/RAM12837/register-app-gitops.git --path . --dest-server https://kubernetes.default.svc --dest-namespace default
argocd app sync register-app
kubectl get pods
kubectl get svc register-app-service
```

After synchronization, copy the application LoadBalancer hostname and browse to `http://<APPLICATION_LOADBALANCER_HOSTNAME>:8080/webapp`. The `/webapp` path is required because the application is packaged as `webapp.war`.

## Jenkins CD Job Configuration

Create a Pipeline job named `gitops-register-app-cd`:

1. Enable **Discard old builds** and keep a maximum of `2` builds.
2. Enable **This project is parameterized** and add String Parameter `IMAGE_TAG`.
3. Configure the GitOps repository URL and the `GitHub` credential.
4. Select **Pipeline script from SCM**, Git, branch `main`, and this repository's `Jenkinsfile`.
5. Save the job so the CI job can trigger it with a value such as `IMAGE_TAG=1.0.0-55`.

The CD pipeline checks out the GitOps repository, replaces the image tag, commits and pushes `deployment.yaml`, and leaves Argo CD to reconcile the new desired state.

## Kubernetes Design

The Deployment selects pods with `app: register-app`, runs two replicas, and exposes container port `8080`. The Service uses the same selector and type `LoadBalancer`, allowing AWS to provision an external endpoint.

## Jenkins Prerequisites

- Jenkins agent label: `Jenkins-Agent`
- A string parameter named `IMAGE_TAG`
- GitHub credential ID: `GitHub`
- Permission for the credential to push to this repository
- Argo CD connected to this repository and configured with the desired sync policy
- An EKS cluster with available worker-node capacity

## GitOps Principle

Jenkins changes Git rather than applying manifests directly to the cluster. Git is the auditable desired state, and Argo CD is responsible for reconciliation. To roll back, revert the image-tag commit and allow Argo CD to synchronize the previous version.

## Security Note

Do not print authenticated Git remote URLs or tokens in Jenkins logs. Use narrowly scoped credentials and prefer a GitHub App or deploy key for long-lived automation.

See the [project README](../README.md), [problems and solutions](../PROBLEMS-AND-SOLUTIONS.md), and [interview questions](../INTERVIEW-QUESTIONS.md).
