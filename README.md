# Vprofile-GitOps-Project-Appside 

This repository is a **follow-up** to [`Vprofile-GitOps-Project-IAC Public`], which provisions the AWS infrastructure (VPC + EKS). Once the cluster is ready, this repo handles CI/CD, SonarCloud analysis, image build & push to ECR, and deployment to EKS.

CI/CD & deployment for the **VProfile** application.  
This repo builds the app image, runs quality checks in **SonarCloud**, pushes to **Amazon ECR**, and deploys to **Amazon EKS** using **Helm** (to the cluster created by `Vprofile-GitOps-Project-IAC Public`).

> The workflow is **manual** (triggered via *workflow_dispatch*) and organized into 3 jobs: **Testing → Build & Publish → DeployToEKS**.

---

## What I did (high level)

1. **Created SonarCloud org & project**, generated a token, and stored credentials as repo secrets.
2. **Updated Kubernetes manifests** under `kubernetes/vpro-app/` and mirrored them into a Helm chart at `helm/vprofilecharts/`.
3. **Added GitHub Actions workflow** at `.github/workflows/main.yml` with a **manual trigger**.
4. Installed **Helm**, created a chart scaffold, **replaced chart templates** with the app’s Kubernetes YAMLs, and committed the chart.
5. Ran the workflow manually to: test + scan, **build & push** to ECR, then **deploy** to EKS.
6. Pointed my domain to the AWS Load Balancer **CNAME** shown by the Ingress.
7. For cleanup: **uninstall ingress controller** first (if you installed one here), then run `terraform destroy` from the infra repo.

---

## Repository layout

```
.
├─ .github/workflows/
│  └─ main.yml                     # Manual pipeline: test+scan → build+push ECR → helm deploy to EKS
├─ helm/vprofilecharts/            # Helm chart (templates are the app’s Kubernetes YAMLs)
│  ├─ Chart.yaml
│  ├─ values.yaml
│  └─ templates/
│     ├─ app-secret.yml
│     ├─ db-CIP.yml
│     ├─ mc-CIP.yml
│     ├─ mcdep.yml
│     ├─ rmq-CIP-service.yml
│     ├─ rmq-dep.yml
│     ├─ vproapp-service.yml
│     ├─ vproappdep.yml
│     ├─ vprodbdep.yml
│     └─ vproingress.yaml
├─ kubernetes/vpro-app/            # Original Kubernetes manifests (mirrored into the chart above)
├─ Dockerfile                      # Multi-stage: Maven build → Tomcat image with WAR
├─ pom.xml, src/                   # Java/Maven app sources
└─ (other: ansible/, Jenkinsfile, etc. not used by this workflow)
```

---

## GitHub secrets required (this repo)

| Secret name              | Purpose |
|--------------------------|---------|
| `AWS_ACCESS_KEY_ID`      | AWS auth for ECR/EKS operations |
| `AWS_SECRET_ACCESS_KEY`  | Pair for the above |
| `REGISTRY`               | **Full ECR registry URI** (e.g. `123456789012.dkr.ecr.us-east-1.amazonaws.com`) |
| `SONAR_TOKEN`            | SonarCloud token for analysis |
| `SONAR_ORGANIZATION`     | SonarCloud organization key |
| `SONAR_PROJECT_KEY`      | SonarCloud project key |
| `SONAR_URL`              | SonarCloud server URL (e.g. `https://sonarcloud.io`) |

### Built-in environment defaults (from the workflow)

- `AWS_REGION`: `us-east-1`  
- `ECR_REPOSITORY`: `vprofileapp`  
- `EKS_CLUSTER`: `vprofile-eks`

> Adjust these in `.github/workflows/main.yml` if your region/cluster/repository differ.

---

## The workflow (`.github/workflows/main.yml`)

**Trigger:** Manual only (`on: workflow_dispatch`). No push/PR triggers by design.

### Job: `Testing`
- Checkout code
- **Maven tests**: `mvn test`
- **Checkstyle**: `mvn checkstyle:checkstyle`
- Set up **Java 11**
- **SonarCloud analysis** using `warchant/setup-sonar-scanner@v7` and the following key parameters:
  - `sonar.host.url=${{ secrets.SONAR_URL }}`
  - `sonar.login=${{ secrets.SONAR_TOKEN }}`
  - `sonar.organization=${{ secrets.SONAR_ORGANIZATION }}`
  - `sonar.projectKey=${{ secrets.SONAR_PROJECT_KEY }}`
  - `sonar.sources=src/`
  - `sonar.junit.reportsPath=target/surefire-reports/`
  - `sonar.jacoco.reportsPath=target/jacoco.exec`
  - `sonar.java.checkstyle.reportPaths=target/checkstyle-result.xml`
  - `sonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/`

> Tip: If you switch to the modern JaCoCo XML report, set `sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml` instead.

### Job: `BUILD_AND_PUBLISH` (needs: Testing)
- Checkout code
- **Build & push Docker image to ECR** via `appleboy/docker-ecr-action@master` using:
  - `registry: ${{ secrets.REGISTRY }}` (your ECR URI)
  - `repo: ${{ env.ECR_REPOSITORY }}` (e.g., `vprofileapp`)
  - `region: ${{ env.AWS_REGION }}`
  - `tags: latest,${{ github.run_number }}`  
  The image ends up as:  
  `REGISTRY/ECR_REPOSITORY:latest` and `REGISTRY/ECR_REPOSITORY:${{ github.run_number }}`

### Job: `DeployToEKS` (needs: BUILD_AND_PUBLISH)
- Checkout code
- (Optionally) configure AWS credentials
- **Helm deploy to EKS** using `bitovi/github-actions-deploy-eks-helm@v1.2.8`:
  - `cluster-name: ${{ env.EKS_CLUSTER }}`
  - `chart-path: helm/vprofilecharts`
  - `namespace: default`
  - `values: appimage=${{ secrets.REGISTRY }}/${{ env.ECR_REPOSITORY }}, apptag=${{ github.run_number }}`
  - `name: vprofile-stack`

> Note: The chart’s `templates/` intentionally contains the original app YAMLs; Helm is used as a simple, repeatable deploy wrapper.

---

## Running the pipeline (manual)

1. Push your changes to the `vprofile-action` repo.
2. In GitHub → **Actions** → select **“vprofile actions”** → **Run workflow** → pick branch → **Run**.
3. Wait for the three jobs: **Testing → Build & Publish → DeployToEKS** to complete.
4. Verify on the cluster:
   ```bash
   kubectl get pods
   kubectl get svc my-app
   kubectl get ingress vpro-ingress
   ```
5. Copy the **ALB DNS** from the Ingress and add a **CNAME** in your domain DNS pointing to it.

---

## Notes on Kubernetes manifests

The app uses multiple components (DB, cache, RMQ, app) with their own Deployments/Services and an Ingress:
- Secrets: `app-secret.yml` (e.g., `db-pass`, `rmq-pass`)
- Deployments/Services: `vproappdep.yml`, `vproapp-service.yml` and others
- Ingress: `vproingress.yaml` (exposes the app via an AWS Load Balancer)

> If you need a public entrypoint, ensure an **Ingress Controller** (e.g., **ingress-nginx**) exists in the cluster. In your infra repo (`iac-vprofile`), this may already be installed by its workflow. If not, install it before deploying the app.

---

## Local build (optional)

To test the image locally:
```bash
# Build
docker build -t vprofileapp:local .

# Run
docker run -p 8080:8080 vprofileapp:local
# Open http://localhost:8080
```

---

## Cleanup

1. If you installed an Ingress Controller as part of app deploys, uninstall it first to release cloud load balancers.
2. Remove the Helm release:
   ```bash
   helm uninstall vprofile-stack -n default
   ```
3. (Optional) delete ECR images you no longer need.
4. Tear down infra from the **`iac-vprofile`** repo:
   ```bash
   terraform destroy
   ```

---
## Credits

- Original app: **/hkhcoder/vprofile-project** sample (Java/Spring, MySQL, Memcached, RabbitMQ)
- Vagrant multi-VM lab & documentation: your implementation (this repo)