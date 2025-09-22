# tg-miniapp-infra

Infrastructure as Code (IaC), Docker, CI/CD, and observability baselines for the Telegram Mini App platform. This repo standardizes how we **build**, **deploy**, and **monitor** every service.

---

## 1) Repo Story

**tg-miniapp-infra** provides reusable Helm charts, Terraform modules, GitHub Actions, and observability templates (OpenTelemetry Collector, Grafana dashboards, Prometheus alerts). It aims for boring, repeatable deployments with strong defaults (TLS, resource requests, probes, structured logs).

---

## 2) Duties & Scope

### Owns

* **Helm umbrella chart** (`charts/miniapp/`) parameterizing each service
* **Terraform** modules for K8s cluster, Redis/Mongo/Postgres, buckets/CDN
* **GitHub Actions** reusable workflows for build → scan → push → deploy
* **Observability**: OTel collector, Prometheus, Loki, Grafana dashboards
* **Security**: Base network policies, PodSecurity, image scanning hooks

### Does **not** own

* Service code (lives in each repo)
* Secret values (live in secret manager)

---

## 3) Layout

```
charts/
  miniapp/
    Chart.yaml
    values.yaml
    templates/
      _helpers.tpl
      deployment.yaml
      service.yaml
      ingress.yaml
      hpa.yaml
      configmap.yaml
      secret.yaml
      servicemonitor.yaml
workflows/
  build-and-deploy.yaml     # GH Actions reusable workflow
terraform/
  modules/
    k8s/
    redis/
    mongo/
    postgres/
    bucket_cdn/
observability/
  otel-collector.yaml
  grafana/
    dashboards/*.json
  prometheus/
    rules/*.yaml
  loki/
    values.yaml
```

---

## 4) Helm — values example

```yaml
image:
  repository: ghcr.io/yourorg/SERVICE_NAME
  tag: "v0.1.0"
replicaCount: 2
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
env:
  - name: NODE_ENV
    value: production
  - name: CONFIG_BASE_URL
    value: https://cdn.example.com/tg-config/prod/current.json
service:
  port: 8080
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: SERVICE_NAME.tg-miniapp.example.com
      paths: ["/"]
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
otel:
  enabled: true
  serviceName: SERVICE_NAME
probes:
  liveness: /healthz
  readiness: /readyz
networkPolicy:
  enabled: true
```

---

## 5) GitHub Actions — reusable workflow

`.github/workflows/build-and-deploy.yaml`

```yaml
name: Build & Deploy
on:
  workflow_call:
    inputs:
      image_name: { required: true, type: string }
      chart_release: { required: true, type: string }
      chart_values: { required: true, type: string }
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with: { registry: ghcr.io, username: ${{ github.actor }}, password: ${{ secrets.GITHUB_TOKEN }} }
      - name: Build & push
        run: |
          docker build -t ghcr.io/${{ github.repository_owner }}/$IMAGE_NAME:$GITHUB_SHA .
          docker push ghcr.io/${{ github.repository_owner }}/$IMAGE_NAME:$GITHUB_SHA
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: azure/setup-kubectl@v4
      - uses: azure/setup-helm@v4
      - name: Helm upgrade
        run: |
          helm upgrade --install $CHART_RELEASE charts/miniapp \
            -f $CHART_VALUES \
            --set image.repository=ghcr.io/${{ github.repository_owner }}/$IMAGE_NAME \
            --set image.tag=$GITHUB_SHA
```

> Each service repo can call this workflow with its own values file.

---

## 6) Observability

* **OpenTelemetry Collector** to receive traces/metrics/logs; export to Tempo/Prometheus/Loki.
* **Grafana dashboards** for latency, error rate, queue depth, settlement SLOs.
* **Prometheus alert rules**: p95 latency thresholds, queue backlog, pod restarts.

---

## 7) Secrets & Security

* Kubernetes Secrets sourced from your secret manager (AWS/GCP/Azure)
* **NetworkPolicies**: default‑deny; allow only needed east‑west traffic
* **Image scanning**: Trivy step in CI; block on critical vulns
* **PodSecurity**: runAsNonRoot, read‑only FS, drop capabilities

---

## 8) Getting Started

1. Pick a cloud (EKS/GKE/AKS) and provision with `terraform/modules/k8s`.
2. Install Redis, Mongo, Postgres modules as needed per environment.
3. Configure **DNS & TLS** for `*.tg-miniapp.example.com`.
4. For each service, prepare a `values-<env>.yaml` and call the reusable workflow.

---

## 9) Testing & Validation

* `helm template` and `helm lint` on PRs
* Terraform plan in CI with manual approval
* Kubeval/Kubeconform for rendered manifests

---

## 10) Roadmap

* GitOps example with ArgoCD app-of-apps
* Multi‑region active‑active blue/green
* Canary analysis with Kayenta/Flagger
