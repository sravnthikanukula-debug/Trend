Trend App — Production Deployment with Terraform, Jenkins CI/CD & Monitoring
Overview

This project deploys the Trend app (a static Vite-built e-commerce storefront) to a production AWS environment: Dockerized, pushed to DockerHub, with infrastructure provisioned via Terraform (VPC, IAM, EC2 running Jenkins), deployed to an EKS cluster, fully automated through a Jenkins declarative pipeline triggered by GitHub webhooks, and monitored with Prometheus + Grafana.

Key links & identifiers
ItemValue
GitHub repositoryhttps://github.com/sravnthikanukula-debug/Trend
DockerHub repositorydocker.io/sravanthikanukuladocker/trend-app
EKS cluster nametrend-cluster (region: us-west-2)
Jenkins URLhttp://34.222.116.152:8080
Jenkins Pipeline jobtrend-app-pipeline
Application URL (LoadBalancer DNS)http://a3e984f8794d54b79ba1ea043833c19b-1097038078.us-west-2.elb.amazonaws.com
Application LoadBalancer ARNarn:aws:elasticloadbalancing:us-west-2:889526028237:loadbalancer/a3e984f8794d54b79ba1ea043833c19b
Grafana URL (monitoring)http://a4d478304f6614dec9ec8c29b82eaa6e-710390400.us-west-2.elb.amazonaws.com
Prometheus URLhttp://a7b0c43ba31a543cf8ee14a090871a5d-1639422963.us-west-2.elb.amazonaws.com:9090
Architecture / pipeline flow
Developer pushes to GitHub (main branch)
        │
        ▼
GitHub Webhook → http://34.222.116.152:8080/github-webhook/
        │
        ▼
Jenkins Pipeline job "trend-app-pipeline" auto-triggers
        │
        ├─ Checkout: pulls source from GitHub (github-creds)
        ├─ Build Docker Image: docker build (serves pre-built dist/ via `serve` on port 3000)
        ├─ Push to DockerHub: docker login + push (dockerhub-creds), tagged with build number + latest
        └─ Deploy to EKS:
             ├─ aws eks update-kubeconfig (via EC2 instance's IAM role)
             ├─ sed-replaces image tag in k8s/deployment.yaml
             ├─ kubectl apply -f k8s/deployment.yaml, k8s/service.yaml
             └─ kubectl rollout status (confirms successful rollout)
        │
        ▼
EKS cluster (trend-cluster, 2 nodes) — 2 pod replicas
        │
        ▼
Kubernetes Service (type: LoadBalancer) → AWS ELB → public internet

Infrastructure for Jenkins itself (VPC, subnet, IGW, route table, security group, IAM role/ instance profile, EC2 instance) is provisioned via Terraform (main.tf), with Jenkins and all required CLI tools (Docker, AWS CLI v2, kubectl, eksctl) installed via an EC2 user-data script.

Setup steps performed
Version control — cloned the source repo, added .gitignore/.dockerignore, pushed to the team's own GitHub repo via CLI (HTTPS + Personal Access Token auth).
Docker — Dockerfile serves the app's pre-built dist/ folder (Vite static output) via serve on port 3000. Built and verified locally in CloudShell.
DockerHub — repository trend-app created; image built and pushed manually first (v1), then automatically via the pipeline on every build (tagged with Jenkins build number + latest).
Terraform — main.tf defines a VPC, public subnet, internet gateway, route table, security group (ports 22/8080/3000), an IAM role + instance profile with admin access, and an EC2 instance running Jenkins, Docker, AWS CLI v2, kubectl, and eksctl (installed via user-data script). Provisioned with terraform init / plan / apply.
Kubernetes / EKS — cluster trend-cluster created via eksctl (2× t3.medium managed nodes). k8s/deployment.yaml (2 replicas, resource limits, readiness/liveness probes) and k8s/service.yaml (type: LoadBalancer) written, applied, and verified working manually before wiring into the pipeline.
Jenkins — installed on the Terraform-provisioned EC2 instance; Docker, Git, Kubernetes, and Pipeline plugins installed. Credentials added for GitHub (github-creds) and DockerHub (dockerhub-creds). A Pipeline job (trend-app-pipeline) created, pointing at the repo's Jenkinsfile via "Pipeline script from SCM."
GitHub webhook — added at Settings → Webhooks on the GitHub repo, pointing to http://34.222.116.152:8080/github-webhook/, triggering the Jenkins job automatically on every push to main.
Monitoring — Prometheus + Grafana (kube-prometheus-stack) installed via Helm into the monitoring namespace, exposed via LoadBalancer services. Pre-built dashboards (Kubernetes Compute Resources, Node Exporter, etc.) confirmed showing live cluster metrics.
Notable issues encountered & fixes
Pre-built static app: the repo ships a Vite dist/ folder, not raw source — Dockerfile simply serves dist/ via serve, no build step needed.
Jenkins install failures (chain of three issues):
The Jenkins APT repo's signing key had expired — worked around by downloading the .deb package directly instead of relying on the signed APT repository.
The downloaded Jenkins version (2.580.1) required Java 21, but only Java 17 was installed — fixed by installing openjdk-21-jdk and switching the default via update-alternatives.
The original Terraform user-data script used set -e and aborted entirely at the GPG key step, so Docker, AWS CLI, kubectl, and eksctl — all meant to install after Jenkins in the same script — never got installed. Fixed by manually installing each tool via SSH post-launch.
Pipeline kubectl apply failing with "the server has asked for the client to provide credentials": the EKS cluster uses the newer Access Entry authentication system (API_AND_CONFIG_MAP mode). The Jenkins EC2 instance's IAM role (trend-jenkins-role) had to be explicitly registered via aws eks create-access-entry / aws eks associate-access-policy (AmazonEKSClusterAdminPolicy) before kubectl commands from Jenkins could authenticate.
sed: can't read k8s/deployment.yaml: the k8s/ manifests were written locally but not committed/pushed in an earlier step — fixed by committing and pushing the k8s/ folder.
How to verify the deployment yourself
bash
kubectl get nodes
kubectl get pods
kubectl get svc trend-app-svc
curl -I http://a3e984f8794d54b79ba1ea043833c19b-1097038078.us-west-2.elb.amazonaws.com

Or simply open the Application URL above in a browser.

To verify monitoring:

bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring

Or open the Grafana URL above (login: admin, password via kubectl get secret --namespace monitoring -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode).

Screenshots

All setup and deployment screenshots (Docker running locally, DockerHub repository, Terraform state/apply output, EKS nodes/pods/service, the app loading via LoadBalancer, the Jenkins dashboard, installed plugins, configured credentials, a successful pipeline run, the GitHub webhook delivery, and the Grafana dashboard showing live cluster metrics) are available here:

https://docs.google.com/document/d/1OJC_ODS7tXcQeF4oj0ZsZFNVCoqAVOWRJk5aapynMl4/view
