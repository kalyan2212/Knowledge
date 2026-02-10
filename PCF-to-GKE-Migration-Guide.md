# PCF to GKE Migration Guide for AppDev Teams

**Version:** 2.0  
**Last Updated:** 2024  
**Audience:** Application Development Teams  
**Platform Owner:** Platform Engineering Team

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current State: PCF Architecture](#2-current-state-pcf-architecture)
3. [Target State: GKE Architecture](#3-target-state-gke-architecture)
4. [Migration Strategy and Phases](#4-migration-strategy-and-phases)
5. [AppDev Team Responsibilities](#5-appdev-team-responsibilities)
6. [Containerization Guide](#6-containerization-guide)
7. [Helm Charts Creation](#7-helm-charts-creation)
8. [PCF to Kubernetes Conversion](#8-pcf-to-kubernetes-conversion)
9. [CI/CD Migration: Jenkins to GitHub Actions](#9-cicd-migration-jenkins-to-github-actions)
10. [Application Code Changes](#10-application-code-changes)
11. [Configuration Management](#11-configuration-management)
12. [Testing and Validation](#12-testing-and-validation)
13. [Rollback Strategies](#13-rollback-strategies)
14. [Best Practices](#14-best-practices)
15. [Troubleshooting](#15-troubleshooting)
16. [Command Reference](#16-command-reference)
17. [Complete End-to-End Example](#17-complete-end-to-end-example)
18. [References](#18-references)

---

## 1. Executive Summary

### 1.1 Migration Overview

This document provides a comprehensive, actionable guide for Application Development (AppDev) teams migrating workloads from **Pivotal Cloud Foundry (PCF)** to **Google Kubernetes Engine (GKE)**. This migration represents a fundamental shift from a Platform-as-a-Service (PaaS) model to a Container-as-a-Service (CaaS) model.

**Migration Scope:**
- Transition from buildpack-based deployments to containerized deployments
- Convert PCF manifest.yml to Kubernetes resources (Deployment, Service, Ingress, ConfigMap, Secret)
- Replace Jenkins+PCF pipeline with unified GitHub Actions workflows
- Adopt Helm for application packaging and deployment
- Implement Kubernetes-native configuration management

### 1.2 Key Transformation Summary

| Aspect | PCF (Current) | GKE (Target) |
|--------|---------------|--------------|
| **Deployment Platform** | Pivotal Cloud Foundry | Google Kubernetes Engine (GKE) |
| **Packaging Method** | Buildpacks (auto-detected) | Container Images (Jib for Java, Docker for Python) |
| **Deployment Tool** | `cf push` | `helm install/upgrade` |
| **Configuration** | manifest.yml + environment variables | Helm values.yaml + ConfigMaps + Secrets |
| **Service Discovery** | PCF internal DNS | Kubernetes Services + DNS |
| **Routing** | PCF Routes | Kubernetes Ingress |
| **CI/CD Pipeline** | GitHub Actions (build) + Jenkins (deploy) | GitHub Actions (build + deploy) |
| **Health Checks** | PCF health checks | Kubernetes liveness/readiness probes |
| **Scaling** | `cf scale` | Horizontal Pod Autoscaler (HPA) |
| **Logging** | PCF loggregator | GKE/Stackdriver logging |
| **Metrics** | PCF metrics | Prometheus + Grafana |

### 1.3 Benefits of Migration

**Technical Benefits:**
- **Flexibility**: Full control over container runtime and dependencies
- **Portability**: Standard containers run anywhere (local, cloud, on-prem)
- **Ecosystem**: Access to vast Kubernetes ecosystem and tooling
- **Scalability**: Advanced scaling options (HPA, VPA, cluster autoscaling)
- **Observability**: Rich metrics, logging, and tracing integrations

**Business Benefits:**
- **Cost Optimization**: Reduced licensing costs, pay for actual resource usage
- **Cloud-Native**: Leverage GCP-managed services and integrations
- **Vendor Neutrality**: Kubernetes is open-source and cloud-agnostic
- **Modern DevOps**: Industry-standard tooling and practices

### 1.4 Migration Timeline

Typical migration timeline per application:
- **Week 1-2**: Assessment and planning
- **Week 3-4**: Containerization and local testing
- **Week 5-6**: Helm chart creation and configuration migration
- **Week 7-8**: CI/CD pipeline migration and integration testing
- **Week 9-10**: UAT and production migration
- **Week 11-12**: Monitoring, optimization, and PCF decommission

### 1.5 Team Responsibilities

**Platform Team (Managed by Platform Engineering):**
- ✅ GKE cluster provisioning and management
- ✅ GitHub Actions runners and enterprise setup
- ✅ Base container images and security scanning
- ✅ Ingress controller and load balancer configuration
- ✅ Monitoring and logging infrastructure (Prometheus, Grafana, Stackdriver)
- ✅ Secret management infrastructure (Google Secret Manager integration)
- ✅ Network policies and service mesh (if applicable)

**AppDev Teams (Your Responsibility):**
- 📋 Application containerization (Jib for Java, Dockerfile for Python)
- 📋 Helm chart creation and maintenance
- 📋 Converting PCF manifest.yml to Kubernetes resources
- 📋 Migrating CI/CD pipelines from Jenkins to GitHub Actions
- 📋 Application code changes (health checks, config, graceful shutdown)
- 📋 Configuration management (ConfigMaps, Secrets)
- 📋 Application-level testing and validation
- 📋 Application monitoring and alerting configuration

---

## 2. Current State: PCF Architecture

### 2.1 How PCF Works

Pivotal Cloud Foundry (PCF), now known as VMware Tanzu Application Service, is a Platform-as-a-Service (PaaS) that provides a high-level abstraction for application deployment.

#### 2.1.1 PCF Deployment Flow

```
Developer Code → cf push → Buildpack Detection → Staging (Droplet Creation) 
    → Instance Deployment → Route Binding → Running Application
```

**Step-by-Step Process:**

1. **Developer Pushes Code**: `cf push my-app -f manifest.yml`
2. **Buildpack Detection**: PCF analyzes code and selects appropriate buildpack
3. **Staging**: 
   - Downloads buildpack
   - Compiles/packages application with dependencies
   - Creates a "droplet" (executable artifact)
4. **Deployment**:
   - PCF schedules instances across Diego cells
   - Starts application instances
   - Monitors health
5. **Routing**:
   - Binds routes to application
   - Load balances traffic across instances

#### 2.1.2 Sample PCF manifest.yml

```yaml
---
applications:
- name: my-java-app
  memory: 1G
  instances: 3
  path: target/my-app.jar
  buildpack: java_buildpack
  routes:
  - route: my-app.apps.pcf-domain.com
  env:
    SPRING_PROFILES_ACTIVE: production
    LOG_LEVEL: INFO
    JBP_CONFIG_OPEN_JDK_JRE: '{ jre: { version: 11.+ } }'
  services:
  - my-postgres-db
  - my-redis-cache
  health-check-type: http
  health-check-http-endpoint: /actuator/health
  timeout: 180
```

#### 2.1.3 PCF Buildpacks

Common buildpacks used:
- **java_buildpack**: For Spring Boot, Tomcat, standalone Java apps
- **python_buildpack**: For Flask, Django, Python apps
- **nodejs_buildpack**: For Node.js and Express apps
- **go_buildpack**: For Go applications
- **binary_buildpack**: For pre-compiled binaries

**Buildpack Features:**
- Automatic dependency detection
- Runtime version management
- Memory calculation and JVM tuning
- APM agent injection
- Security patches

#### 2.1.4 PCF Service Bindings

PCF automatically injects service credentials into applications via environment variables:

```bash
# VCAP_SERVICES environment variable
{
  "postgresql": [{
    "name": "my-postgres-db",
    "credentials": {
      "uri": "postgres://user:pass@host:5432/db",
      "username": "user",
      "password": "password",
      "host": "postgres-host",
      "port": 5432,
      "database": "mydb"
    }
  }]
}
```

Applications parse `VCAP_SERVICES` to extract connection information.

#### 2.1.5 Current CI/CD Architecture

```
GitHub Repository → GitHub Actions (Build & Test) → Artifact Storage 
    → Jenkins (Deploy) → cf push → PCF
```

**GitHub Actions Workflow (Build):**
```yaml
# .github/workflows/build.yml
name: Build and Test
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up JDK 11
        uses: actions/setup-java@v3
        with:
          java-version: '11'
      - name: Build with Maven
        run: mvn clean package
      - name: Run Tests
        run: mvn test
      - name: Upload Artifact
        uses: actions/upload-artifact@v3
        with:
          name: app-jar
          path: target/*.jar
```

**Jenkins Pipeline (Deploy):**
```groovy
pipeline {
    agent any
    stages {
        stage('Download Artifact') {
            steps {
                // Download from artifact repository
            }
        }
        stage('Deploy to PCF') {
            steps {
                sh 'cf login -a ${PCF_API} -u ${PCF_USER} -p ${PCF_PASS}'
                sh 'cf push my-app -f manifest.yml'
            }
        }
    }
}
```

### 2.2 PCF Advantages (Being Lost)

**Simplicity:**
- Single command deployment (`cf push`)
- No need to understand containers or orchestration
- Automatic buildpack selection
- Zero-downtime deployments by default

**Developer Productivity:**
- Fast iteration cycles
- Minimal configuration required
- Built-in service marketplace
- Automatic SSL/TLS certificate management

**Operational Simplicity:**
- Platform handles infrastructure
- Automatic security patches via buildpack updates
- Built-in logging and metrics aggregation
- Simple scaling: `cf scale my-app -i 5`

### 2.3 PCF Limitations (Being Addressed)

**Vendor Lock-In:**
- Proprietary platform with high licensing costs
- Limited portability to other clouds or on-premises
- Tied to VMware/Tanzu ecosystem

**Limited Flexibility:**
- Restricted runtime customization
- Limited control over networking
- Buildpack limitations for custom requirements
- Difficult to run stateful applications

**Scalability Constraints:**
- Limited support for microservices patterns
- No native service mesh support
- Limited control over resource allocation

**Cost:**
- High licensing fees per foundation
- Per-instance pricing model
- Additional costs for services and add-ons

---

## 3. Target State: GKE Architecture

### 3.1 How GKE Works

Google Kubernetes Engine (GKE) is a managed Kubernetes service that provides container orchestration at scale.



#### 3.1.1 GKE Deployment Flow

```
Container Image → Container Registry (GCR/Artifact Registry) → Helm Chart 
    → Kubernetes API → Scheduler → Pods on Nodes → Service → Ingress → Users
```

**Step-by-Step Process:**

1. **Build Container Image**: Using Jib (Java) or Docker (Python)
2. **Push to Registry**: Store image in Google Artifact Registry/GCR  
3. **Deploy with Helm**:
   ```bash
   helm upgrade --install my-app ./helm-chart \
     --set image.tag=v1.2.3 \
     --namespace production
   ```
4. **Kubernetes Orchestration**:
   - API Server validates and stores deployment spec
   - Scheduler assigns pods to nodes
   - Kubelet pulls container images and starts containers
   - Service creates stable endpoint for pod set
   - Ingress configures external load balancer

#### 3.1.2 Core Kubernetes Concepts

**Pods:**
- Smallest deployable unit in Kubernetes
- One or more containers sharing network and storage
- Ephemeral by design (can be replaced at any time)

**Deployments:**
- Manages desired state for pods
- Handles rolling updates and rollbacks
- Ensures specified number of replicas are running

**Services:**
- Stable network endpoint for accessing pods
- Types: ClusterIP (internal), NodePort, LoadBalancer
- Performs load balancing across pod replicas

**ConfigMaps & Secrets:**
- ConfigMaps: Non-sensitive configuration data
- Secrets: Sensitive data (passwords, tokens, certificates)
- Injected as environment variables or mounted volumes

**Ingress:**
- HTTP/HTTPS routing and load balancing
- SSL/TLS termination
- Path-based and host-based routing

**Namespaces:**
- Virtual clusters within physical cluster
- Resource isolation and organization
- Typical: dev, staging, production

### 3.2 Helm Overview

Helm is the package manager for Kubernetes.

**Helm Benefits:**
- **Templating**: Reusable, parameterized Kubernetes manifests
- **Versioning**: Track application releases
- **Rollback**: Easy rollback to previous versions
- **Package Management**: Bundle all K8s resources
- **Configuration Management**: Separate config from templates

---

## 4. Migration Strategy and Phases

### 4.1 Migration Approach

**Recommended Strategy: Strangler Fig Pattern**
- Migrate applications incrementally
- Both PCF and GKE run in parallel
- Route traffic gradually from PCF to GKE
- Decommission PCF apps after validation

### 4.2 Success Criteria

**Technical:**
- ✅ Application running on GKE
- ✅ All tests passing
- ✅ Performance meets baseline
- ✅ CI/CD fully automated

---

## 5. AppDev Team Responsibilities

### 5.1 Task Checklist

1. **Application Assessment** (2-3 days)
   - Review PCF manifest.yml
   - Document dependencies
   - Identify challenges

2. **Containerization** (3-5 days)
   - Java: Add Jib plugin
   - Python: Create Dockerfile
   - Test containers locally

3. **Helm Chart Creation** (5-7 days)
   - Create chart structure
   - Write templates
   - Create values files

4. **CI/CD Migration** (3-5 days)
   - Create GitHub Actions workflow
   - Configure environments
   - Test pipeline

5. **Code Changes** (3-7 days)
   - Implement health checks
   - Update configuration
   - Graceful shutdown

6. **Testing** (5-7 days)
   - Unit, integration, E2E tests
   - Performance testing
   - UAT

---

## 6. Containerization Guide

### 6.1 Java with Jib (Maven)

```xml
<plugin>
  <groupId>com.google.cloud.tools</groupId>
  <artifactId>jib-maven-plugin</artifactId>
  <version>3.4.0</version>
  <configuration>
    <from>
      <image>gcr.io/distroless/java11-debian11:nonroot</image>
    </from>
    <to>
      <image>gcr.io/my-project/my-app</image>
      <tags>
        <tag>${project.version}</tag>
        <tag>latest</tag>
      </tags>
    </to>
    <container>
      <jvmFlags>
        <jvmFlag>-Xms512m</jvmFlag>
        <jvmFlag>-Xmx1024m</jvmFlag>
        <jvmFlag>-XX:+UseG1GC</jvmFlag>
      </jvmFlags>
      <ports>
        <port>8080</port>
      </ports>
      <user>nonroot</user>
    </container>
  </configuration>
</plugin>
```

**Build:**
```bash
mvn clean compile jib:build
```

### 6.2 Java with Jib (Gradle)

```groovy
plugins {
    id 'com.google.cloud.tools.jib' version '3.4.0'
}

jib {
    from {
        image = 'gcr.io/distroless/java11-debian11:nonroot'
    }
    to {
        image = "gcr.io/my-project/my-app"
        tags = [version, 'latest']
    }
    container {
        jvmFlags = ['-Xms512m', '-Xmx1024m', '-XX:+UseG1GC']
        ports = ['8080']
        user = 'nonroot'
    }
}
```

**Build:**
```bash
./gradlew clean jib
```

### 6.3 Python with Docker

```dockerfile
# Multi-stage Dockerfile
FROM python:3.11-slim as builder

WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libpq-dev && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11-slim

RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 curl && rm -rf /var/lib/apt/lists/*

COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appuser . .

ENV PATH=/home/appuser/.local/bin:$PATH \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

USER appuser
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:8080/health || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "4", "app:app"]
```

**Build:**
```bash
docker build -t gcr.io/my-project/my-python-app:v1.0.0 .
docker push gcr.io/my-project/my-python-app:v1.0.0
```

### 6.4 Python App Example

**app.py:**
```python
from flask import Flask, jsonify
import os
import logging

logging.basicConfig(level=os.getenv('LOG_LEVEL', 'INFO'))
app = Flask(__name__)

@app.route('/health')
def health():
    return jsonify({"status": "UP"}), 200

@app.route('/health/ready')
def ready():
    try:
        # Check dependencies
        return jsonify({"status": "READY"}), 200
    except Exception as e:
        return jsonify({"status": "NOT_READY"}), 503

@app.route('/')
def index():
    return jsonify({"message": "Hello from GKE!"}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

---

## 7. Helm Charts Creation

### 7.1 Chart Structure

```
my-app-chart/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-prod.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── hpa.yaml
    ├── _helpers.tpl
    └── NOTES.txt
```

### 7.2 Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: My Application Helm Chart
type: application
version: 1.0.0
appVersion: "1.0.0"
maintainers:
  - name: AppDev Team
    email: team@example.com
```

### 7.3 values.yaml

```yaml
replicaCount: 2

image:
  repository: gcr.io/my-project/my-app
  pullPolicy: IfNotPresent
  tag: ""

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: "gce"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: my-app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: my-app-tls
      hosts:
        - my-app.example.com

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 1Gi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 5

config:
  logLevel: INFO
  springProfilesActive: production

env:
  - name: SPRING_PROFILES_ACTIVE
    value: "production"
  - name: LOG_LEVEL
    value: "INFO"
```

### 7.4 templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
          protocol: TCP
        livenessProbe:
          {{- toYaml .Values.livenessProbe | nindent 12 }}
        readinessProbe:
          {{- toYaml .Values.readinessProbe | nindent 12 }}
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
        env:
          {{- toYaml .Values.env | nindent 12 }}
```

### 7.5 templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "my-app.selectorLabels" . | nindent 4 }}
```

### 7.6 templates/ingress.yaml

```yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "my-app.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

### 7.7 templates/hpa.yaml

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "my-app.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
{{- end }}
```

### 7.8 templates/_helpers.tpl

```yaml
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "my-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{- define "my-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "my-app.labels" -}}
helm.sh/chart: {{ include "my-app.chart" . }}
{{ include "my-app.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### 7.9 Helm Commands

```bash
# Validate chart
helm lint ./my-app-chart

# Dry-run install
helm install my-app ./my-app-chart --dry-run --debug

# Install chart
helm install my-app ./my-app-chart -f values-dev.yaml -n dev

# Upgrade chart
helm upgrade my-app ./my-app-chart -f values-prod.yaml -n production

# Rollback
helm rollback my-app 1 -n production

# List releases
helm list -n production

# Get values
helm get values my-app -n production

# Uninstall
helm uninstall my-app -n dev
```

---

## 8. PCF to Kubernetes Conversion

### 8.1 Mapping Table

| PCF Concept | Kubernetes Equivalent |
|-------------|----------------------|
| name | Deployment metadata.name |
| memory | resources.limits.memory |
| instances | spec.replicas |
| buildpack | Container image |
| routes | Ingress |
| env | env / ConfigMap / Secret |
| services | External + Secrets |
| health-check-http-endpoint | livenessProbe.httpGet.path |
| timeout | initialDelaySeconds |

### 8.2 Example Conversion

**PCF manifest.yml:**
```yaml
applications:
- name: my-app
  memory: 1G
  instances: 3
  routes:
  - route: my-app.apps.pcf.com
  env:
    SPRING_PROFILES_ACTIVE: production
  services:
  - postgres-db
  health-check-type: http
  health-check-http-endpoint: /actuator/health
```

**Kubernetes Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: gcr.io/my-project/my-app:v1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: my-app-secrets
              key: database-url
        resources:
          limits:
            memory: "1Gi"
          requests:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 60
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
spec:
  rules:
  - host: my-app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app
            port:
              number: 80
```

---

## 9. CI/CD Migration: Jenkins to GitHub Actions

### 9.1 Complete GitHub Actions Workflow

```yaml
name: Build and Deploy

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  GCP_PROJECT_ID: my-project
  GKE_CLUSTER: my-cluster
  GKE_ZONE: us-central1-a
  IMAGE_NAME: my-app

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up JDK 11
      uses: actions/setup-java@v4
      with:
        java-version: '11'
        distribution: 'temurin'
        cache: maven
    
    - name: Run tests
      run: mvn test
    
    - name: Build
      run: mvn clean package -DskipTests

  build-container:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    
    permissions:
      contents: read
      id-token: write
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up JDK 11
      uses: actions/setup-java@v4
      with:
        java-version: '11'
        cache: maven
    
    - name: Authenticate to GCP
      uses: google-github-actions/auth@v2
      with:
        workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
        service_account: ${{ secrets.WIF_SERVICE_ACCOUNT }}
    
    - name: Set up Cloud SDK
      uses: google-github-actions/setup-gcloud@v2
    
    - name: Configure Docker for GCR
      run: gcloud auth configure-docker gcr.io
    
    - name: Build and push with Jib
      run: |
        mvn compile jib:build \
          -Djib.to.image=gcr.io/${{ env.GCP_PROJECT_ID }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
    
    outputs:
      image-tag: ${{ github.sha }}

  deploy-dev:
    needs: build-container
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: development
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Authenticate to GCP
      uses: google-github-actions/auth@v2
      with:
        workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
        service_account: ${{ secrets.WIF_SERVICE_ACCOUNT }}
    
    - name: Get GKE credentials
      uses: google-github-actions/get-gke-credentials@v2
      with:
        cluster_name: ${{ env.GKE_CLUSTER }}
        location: ${{ env.GKE_ZONE }}
    
    - name: Install Helm
      uses: azure/setup-helm@v3
      with:
        version: '3.13.0'
    
    - name: Deploy with Helm
      run: |
        helm upgrade --install ${{ env.IMAGE_NAME }} ./helm-chart \
          -n dev \
          --create-namespace \
          --set image.tag=${{ needs.build-container.outputs.image-tag }} \
          -f ./helm-chart/values-dev.yaml \
          --wait --timeout 5m
    
    - name: Verify deployment
      run: kubectl rollout status deployment/${{ env.IMAGE_NAME }} -n dev

  deploy-prod:
    needs: build-container
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Authenticate to GCP
      uses: google-github-actions/auth@v2
      with:
        workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
        service_account: ${{ secrets.WIF_SERVICE_ACCOUNT }}
    
    - name: Get GKE credentials
      uses: google-github-actions/get-gke-credentials@v2
      with:
        cluster_name: ${{ env.GKE_CLUSTER }}
        location: ${{ env.GKE_ZONE }}
    
    - name: Install Helm
      uses: azure/setup-helm@v3
    
    - name: Deploy with Helm
      run: |
        helm upgrade --install ${{ env.IMAGE_NAME }} ./helm-chart \
          -n production \
          --create-namespace \
          --set image.tag=${{ needs.build-container.outputs.image-tag }} \
          -f ./helm-chart/values-prod.yaml \
          --wait --timeout 10m
    
    - name: Smoke test
      run: |
        kubectl run test --rm -i --restart=Never \
          --image=curlimages/curl:latest \
          -n production \
          -- curl -f http://${{ env.IMAGE_NAME }}/health
```

---

## 10. Application Code Changes

### 10.1 Health Checks - Spring Boot

```java
// application.yml
management:
  endpoints:
    web:
      base-path: /actuator
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

**Custom Health Indicator:**
```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    @Autowired
    private DataSource dataSource;
    
    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            if (conn.isValid(1)) {
                return Health.up().build();
            }
        } catch (Exception e) {
            return Health.down(e).build();
        }
        return Health.down().build();
    }
}
```

### 10.2 Health Checks - Python Flask

```python
from flask import Flask, jsonify
import psycopg2
import os

app = Flask(__name__)

@app.route('/health')
def liveness():
    """Liveness probe"""
    return jsonify({"status": "UP"}), 200

@app.route('/health/ready')
def readiness():
    """Readiness probe"""
    try:
        # Check database
        conn = psycopg2.connect(os.getenv('DATABASE_URL'))
        conn.close()
        return jsonify({"status": "READY"}), 200
    except Exception as e:
        return jsonify({"status": "NOT_READY", "error": str(e)}), 503
```

### 10.3 Graceful Shutdown - Spring Boot

```yaml
# application.yml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

### 10.4 Graceful Shutdown - Python

```python
# gunicorn.conf.py
graceful_timeout = 30
timeout = 120

def on_starting(server):
    server.log.info("Starting")

def worker_int(worker):
    worker.log.info("Shutting down gracefully")
```

### 10.5 Configuration - Remove VCAP_SERVICES

**OLD (PCF-specific):**
```java
@Configuration
public class CloudConfig extends AbstractCloudConfig {
    @Bean
    public DataSource dataSource() {
        return connectionFactory().dataSource();
    }
}
```

**NEW (Standard):**
```java
@Configuration
public class DataSourceConfig {
    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create()
            .url(System.getenv("DATABASE_URL"))
            .build();
    }
}
```

---

## 11. Configuration Management

### 11.1 ConfigMap Example

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
  namespace: production
data:
  SPRING_PROFILES_ACTIVE: "production"
  LOG_LEVEL: "INFO"
  DATABASE_POOL_SIZE: "20"
  FEATURE_NEW_UI: "true"
```

### 11.2 Secret Example

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secrets
  namespace: production
type: Opaque
stringData:
  DATABASE_URL: "postgresql://user:pass@host:5432/db"
  DATABASE_PASSWORD: "supersecret"
  API_KEY: "api-key-12345"
```

**Create from CLI:**
```bash
kubectl create secret generic my-app-secrets \
  --from-literal=database-password=supersecret \
  -n production

# Using Google Secret Manager
kubectl create secret generic my-app-secrets \
  --from-literal=db-pass=$(gcloud secrets versions access latest --secret=db-password)
```

---

## 12. Testing and Validation

### 12.1 Testing Checklist

- [ ] Unit tests pass
- [ ] Container builds
- [ ] Container runs locally
- [ ] Helm chart validates
- [ ] Deploy to dev succeeds
- [ ] Health endpoints respond
- [ ] Database connectivity works
- [ ] Load testing passes
- [ ] UAT approved

### 12.2 Testing Commands

```bash
# Test container locally
docker run -p 8080:8080 my-app:latest
curl http://localhost:8080/health

# Test in K8s
kubectl run test --rm -i --image=curlimages/curl -- curl my-app/health

# Port-forward
kubectl port-forward svc/my-app 8080:80 -n dev

# Load test
ab -n 1000 -c 10 http://my-app.example.com/
```

---

## 13. Rollback Strategies

### 13.1 Helm Rollback

```bash
# List history
helm history my-app -n production

# Rollback to previous
helm rollback my-app -n production

# Rollback to specific revision
helm rollback my-app 3 -n production
```

### 13.2 Kubernetes Rollback

```bash
# Check status
kubectl rollout status deployment/my-app -n production

# View history
kubectl rollout history deployment/my-app -n production

# Rollback to previous
kubectl rollout undo deployment/my-app -n production

# Rollback to specific revision
kubectl rollout undo deployment/my-app --to-revision=2 -n production
```

---

## 14. Best Practices

### 14.1 Container Best Practices

- ✅ Use non-root user
- ✅ Use minimal base images
- ✅ Scan for vulnerabilities
- ✅ Don't include secrets
- ✅ Use multi-stage builds
- ✅ Log to STDOUT/STDERR

### 14.2 Kubernetes Best Practices

- ✅ Set resource limits
- ✅ Implement health probes
- ✅ Use namespaces
- ✅ Use ConfigMaps/Secrets
- ✅ Enable autoscaling
- ✅ Use proper labels

### 14.3 Helm Best Practices

- ✅ Version your charts
- ✅ Use values files per environment
- ✅ Template all configurable values
- ✅ Test with helm lint
- ✅ Document values

---

## 15. Troubleshooting

### 15.1 Common Issues

**Pod in CrashLoopBackOff:**
```bash
kubectl logs my-app-pod -n production
kubectl describe pod my-app-pod -n production
kubectl logs my-app-pod --previous -n production
```

**Image Pull Errors:**
```bash
gcloud container images list --repository=gcr.io/my-project
kubectl describe pod my-app-pod -n production
```

**Probe Failures:**
```bash
kubectl run test --rm -i --image=curlimages/curl -- curl my-app:80/health
# Increase initialDelaySeconds if needed
```

---

## 16. Command Reference

### 16.1 kubectl Commands

```bash
# Get resources
kubectl get pods -n production
kubectl get deployments -n production
kubectl get services -n production
kubectl get ingress -n production

# Logs
kubectl logs -f deployment/my-app -n production

# Execute
kubectl exec -it my-app-pod -n production -- /bin/sh

# Port forward
kubectl port-forward svc/my-app 8080:80 -n production

# Scale
kubectl scale deployment my-app --replicas=5 -n production
```

### 16.2 Helm Commands

```bash
# Install/Upgrade
helm install my-app ./chart -n production
helm upgrade my-app ./chart -n production
helm upgrade --install my-app ./chart -n production

# List
helm list -n production

# Rollback
helm rollback my-app -n production

# Uninstall
helm uninstall my-app -n production
```

### 16.3 Docker/Jib Commands

```bash
# Docker
docker build -t my-app:latest .
docker push gcr.io/my-project/my-app:v1.0.0

# Jib Maven
mvn compile jib:build

# Jib Gradle
./gradlew jib
```

---

## 17. Complete End-to-End Example

### 17.1 Sample Migration

**PCF manifest.yml:**
```yaml
applications:
- name: sample-app
  memory: 1G
  instances: 2
  routes:
  - route: sample-app.pcf.com
  env:
    SPRING_PROFILES_ACTIVE: production
  services:
  - postgres-db
```

**Step 1: Add Jib to pom.xml**
```xml
<plugin>
  <groupId>com.google.cloud.tools</groupId>
  <artifactId>jib-maven-plugin</artifactId>
  <version>3.4.0</version>
</plugin>
```

**Step 2: Create Helm chart**
```bash
helm create sample-app-chart
```

**Step 3: Create GitHub Actions**
```yaml
name: Deploy
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: mvn compile jib:build
    - run: helm upgrade --install sample-app ./chart
```

**Step 4: Deploy**
```bash
git push origin main
```

---

## 18. References

### 18.1 Official Documentation

**GCP & GKE:**
- GKE Docs: https://cloud.google.com/kubernetes-engine/docs
- GKE Best Practices: https://cloud.google.com/kubernetes-engine/docs/best-practices

**Kubernetes:**
- K8s Docs: https://kubernetes.io/docs/
- kubectl Cheat Sheet: https://kubernetes.io/docs/reference/kubectl/cheatsheet/

**Helm:**
- Helm Docs: https://helm.sh/docs/
- Chart Best Practices: https://helm.sh/docs/chart_best_practices/

**Jib:**
- Jib GitHub: https://github.com/GoogleContainerTools/jib
- Jib Maven: https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin
- Jib Gradle: https://github.com/GoogleContainerTools/jib/tree/master/jib-gradle-plugin

**Docker:**
- Docker Docs: https://docs.docker.com/
- Dockerfile Best Practices: https://docs.docker.com/develop/dev-best-practices/

**GitHub Actions:**
- Actions Docs: https://docs.github.com/en/actions
- Workflow Syntax: https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions

---

## Appendix A: PCF vs GKE Commands

| Task | PCF | GKE/Kubernetes |
|------|-----|----------------|
| Deploy | `cf push` | `helm upgrade --install` |
| Scale | `cf scale my-app -i 5` | `kubectl scale deployment my-app --replicas=5` |
| Logs | `cf logs my-app` | `kubectl logs -f deployment/my-app` |
| Restart | `cf restart my-app` | `kubectl rollout restart deployment/my-app` |
| SSH | `cf ssh my-app` | `kubectl exec -it my-app-pod -- /bin/sh` |
| Set env | `cf set-env` | `kubectl set env deployment/my-app KEY=value` |
| Info | `cf app my-app` | `kubectl describe deployment my-app` |
| Delete | `cf delete my-app` | `helm uninstall my-app` |

---

## Appendix B: Glossary

- **Container**: Package with app code and dependencies
- **Pod**: Smallest K8s unit (1+ containers)
- **Deployment**: Manages desired state for pods
- **Service**: Stable network endpoint for pods
- **Ingress**: HTTP/HTTPS routing
- **Helm**: K8s package manager
- **Jib**: Container builder for Java (no Docker)
- **GKE**: Google Kubernetes Engine
- **ConfigMap**: Non-sensitive config
- **Secret**: Sensitive data
- **HPA**: Horizontal Pod Autoscaler
- **Probe**: Health check (liveness/readiness)
- **Namespace**: Virtual cluster for isolation

---

**Document Status:** Complete ✅  
**Last Review:** 2024  
**Next Review:** Quarterly  

**Contact:**
- Platform Team: platform@example.com
- Slack: #gke-migration
- Portal: https://migration.example.com
