# 数据中台系统CI/CD流水线文档

## 文档信息

| 项目 | 内容 |
|------|------|
| 文档名称 | 数据中台系统CI/CD流水线文档 |
| 文档版本 | V1.0 |
| 编制日期 | 2026-03-31 |
| 编制人 | DevOps组 |
| 审批状态 | 待审批 |

---

## 1. CI/CD概述

### 1.1 目标

- **持续集成**：代码提交后自动构建、测试
- **持续交付**：自动部署到测试环境
- **持续部署**：自动部署到生产环境
- **质量门禁**：代码质量检查、测试覆盖率

### 1.2 工具链

| 环节 | 工具 | 用途 |
|------|------|------|
| 代码管理 | GitLab/GitHub | 代码仓库 |
| CI/CD | GitLab CI/GitHub Actions | 流水线执行 |
| 镜像仓库 | Harbor | Docker镜像存储 |
| 制品仓库 | Nexus | 依赖包管理 |
| 代码质量 | SonarQube | 代码扫描 |
| 安全扫描 | Trivy | 镜像漏洞扫描 |
| 部署 | ArgoCD | GitOps部署 |

---

## 2. 流水线设计

### 2.1 流水线流程

```
代码提交 → 代码检查 → 构建 → 测试 → 安全扫描 → 打包 → 部署
    │         │        │      │         │        │       │
    ▼         ▼        ▼      ▼         ▼        ▼       ▼
 触发流水线  Lint    编译    单元测试   漏洞扫描   镜像    Dev
  自动执行   静态分析  打包    集成测试   依赖检查   推送   自动部署
```

### 2.2 分支策略

| 分支 | 触发条件 | 部署环境 | 流水线 |
|------|---------|---------|--------|
| feature/* | PR创建 | - | 代码检查+单元测试 |
| develop | 代码推送 | 开发环境 | 完整CI+自动部署 |
| release/* | 代码推送 | 测试环境 | 完整CI+自动部署 |
| main | 代码推送 | 预发布环境 | 完整CI+自动部署 |
| tag | 标签推送 | 生产环境 | 完整CI+手动审批部署 |

---

## 3. GitLab CI配置

### 3.1 后端CI配置

```yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - build
  - security
  - deploy

variables:
  DOCKER_REGISTRY: "registry.example.com"
  PROJECT_NAME: "dmp-backend"
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

cache:
  paths:
    - .cache/pip/
    - venv/

# ========== Lint阶段 ==========
flake8:
  stage: lint
  image: python:3.11-slim
  before_script:
    - pip install flake8
  script:
    - flake8 apps/ --max-line-length=120 --exclude=venv
  allow_failure: false

black:
  stage: lint
  image: python:3.11-slim
  before_script:
    - pip install black
  script:
    - black --check apps/
  allow_failure: false

isort:
  stage: lint
  image: python:3.11-slim
  before_script:
    - pip install isort
  script:
    - isort --check-only apps/
  allow_failure: false

mypy:
  stage: lint
  image: python:3.11-slim
  before_script:
    - pip install mypy
    - pip install -r requirements.txt
  script:
    - mypy apps/ --ignore-missing-imports
  allow_failure: true

# ========== Test阶段 ==========
unit-test:
  stage: test
  image: python:3.11-slim
  services:
    - postgres:16-alpine
    - redis:7-alpine
  variables:
    POSTGRES_DB: test
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    DATABASE_URL: postgresql://test:test@postgres/test
    REDIS_URL: redis://redis:6379/0
  before_script:
    - pip install -r requirements.txt
    - pip install -r requirements-dev.txt
  script:
    - pytest tests/unit/ -v --cov=apps --cov-report=xml --cov-report=html
  coverage: '/TOTAL.+ ([0-9]{1,3}%)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
    paths:
      - htmlcov/
    expire_in: 1 week

integration-test:
  stage: test
  image: python:3.11-slim
  services:
    - postgres:16-alpine
    - redis:7-alpine
    - minio/minio:latest
  variables:
    POSTGRES_DB: test
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    DATABASE_URL: postgresql://test:test@postgres/test
    REDIS_URL: redis://redis:6379/0
  before_script:
    - pip install -r requirements.txt
    - pip install -r requirements-dev.txt
  script:
    - pytest tests/integration/ -v
  only:
    - develop
    - merge_requests

# ========== Build阶段 ==========
build-image:
  stage: build
  image: docker:24-dind
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $DOCKER_REGISTRY
  script:
    - docker build 
        --build-arg VERSION=$CI_COMMIT_SHA
        -t $DOCKER_REGISTRY/dmp/backend:$CI_COMMIT_SHA
        -t $DOCKER_REGISTRY/dmp/backend:latest
        .
    - docker push $DOCKER_REGISTRY/dmp/backend:$CI_COMMIT_SHA
    - docker push $DOCKER_REGISTRY/dmp/backend:latest
  only:
    - develop
    - main
    - tags

# ========== Security阶段 ==========
sonarqube:
  stage: security
  image: sonarsource/sonar-scanner-cli:latest
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"
  cache:
    key: "${CI_JOB_NAME}"
    paths:
      - .sonar/cache
  script:
    - sonar-scanner
        -Dsonar.projectKey=dmp-backend
        -Dsonar.sources=apps/
        -Dsonar.python.coverage.reportPaths=coverage.xml
        -Dsonar.qualitygate.wait=true
  allow_failure: false

trivy-scan:
  stage: security
  image: aquasec/trivy:latest
  script:
    - trivy image 
        --severity HIGH,CRITICAL
        --exit-code 1
        $DOCKER_REGISTRY/dmp/backend:$CI_COMMIT_SHA
  allow_failure: true
  only:
    - develop
    - main
    - tags

# ========== Deploy阶段 ==========
deploy-dev:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context dev
    - kubectl set image deployment/dmp-integration-service
        dmp-integration-service=$DOCKER_REGISTRY/dmp/backend:$CI_COMMIT_SHA
        -n dmp-dev
    - kubectl rollout status deployment/dmp-integration-service -n dmp-dev --timeout=300s
  environment:
    name: development
    url: https://dev.dmp.example.com
  only:
    - develop

deploy-staging:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context staging
    - kubectl set image deployment/dmp-integration-service
        dmp-integration-service=$DOCKER_REGISTRY/dmp/backend:$CI_COMMIT_SHA
        -n dmp-staging
    - kubectl rollout status deployment/dmp-integration-service -n dmp-staging --timeout=300s
  environment:
    name: staging
    url: https://staging.dmp.example.com
  only:
    - main

deploy-prod:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context prod
    - kubectl set image deployment/dmp-integration-service
        dmp-integration-service=$DOCKER_REGISTRY/dmp/backend:$CI_COMMIT_TAG
        -n dmp-prod
    - kubectl rollout status deployment/dmp-integration-service -n dmp-prod --timeout=300s
  environment:
    name: production
    url: https://dmp.example.com
  when: manual
  only:
    - tags
```

### 3.2 前端CI配置

```yaml
# .gitlab-ci.yml (Frontend)
stages:
  - lint
  - test
  - build
  - deploy

variables:
  DOCKER_REGISTRY: "registry.example.com"
  NODE_VERSION: "20"

# ========== Lint阶段 ==========
eslint:
  stage: lint
  image: node:$NODE_VERSION-alpine
  cache:
    paths:
      - node_modules/
  before_script:
    - npm ci
  script:
    - npm run lint

type-check:
  stage: lint
  image: node:$NODE_VERSION-alpine
  cache:
    paths:
      - node_modules/
  before_script:
    - npm ci
  script:
    - npm run type-check

# ========== Test阶段 ==========
unit-test:
  stage: test
  image: node:$NODE_VERSION-alpine
  cache:
    paths:
      - node_modules/
  before_script:
    - npm ci
  script:
    - npm run test:unit -- --coverage
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

# ========== Build阶段 ==========
build:
  stage: build
  image: node:$NODE_VERSION-alpine
  cache:
    paths:
      - node_modules/
  before_script:
    - npm ci
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 week

build-image:
  stage: build
  image: docker:24-dind
  services:
    - docker:24-dind
  dependencies:
    - build
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $DOCKER_REGISTRY
  script:
    - docker build 
        -t $DOCKER_REGISTRY/dmp/frontend:$CI_COMMIT_SHA
        -t $DOCKER_REGISTRY/dmp/frontend:latest
        .
    - docker push $DOCKER_REGISTRY/dmp/frontend:$CI_COMMIT_SHA
    - docker push $DOCKER_REGISTRY/dmp/frontend:latest
  only:
    - develop
    - main
    - tags

# ========== Deploy阶段 ==========
deploy-dev:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context dev
    - kubectl set image deployment/dmp-frontend
        dmp-frontend=$DOCKER_REGISTRY/dmp/frontend:$CI_COMMIT_SHA
        -n dmp-dev
  environment:
    name: development
    url: https://dev.dmp.example.com
  only:
    - develop
```

---

## 4. ArgoCD配置

### 4.1 应用配置

```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dmp-backend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/dmp-gitops.git
    targetRevision: HEAD
    path: overlays/production/backend
  destination:
    server: https://kubernetes.default.svc
    namespace: dmp-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### 4.2 Kustomize配置

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: dmp-prod

resources:
  - ../../base

images:
  - name: dmp-backend
    newName: registry.example.com/dmp/backend
    newTag: v1.0.0

patchesStrategicMerge:
  - deployment-patch.yaml
  - hpa-patch.yaml

configMapGenerator:
  - name: dmp-config
    behavior: merge
    literals:
      - APP_ENV=production
      - LOG_LEVEL=INFO
```

---

## 5. 质量门禁

### 5.1 代码质量指标

| 指标 | 阈值 | 说明 |
|------|------|------|
| 代码覆盖率 | ≥80% | 单元测试覆盖率 |
| 代码重复率 | ≤3% | 重复代码比例 |
| 代码复杂度 | ≤10 | 圈复杂度 |
| 代码异味 | 0 blocker/critical | SonarQube检测 |
| 安全漏洞 | 0 high/critical | 安全扫描 |

### 5.2 质量门禁配置

```yaml
# .sonarqube/quality-gate.yaml
conditions:
  - metric: coverage
    operator: LT
    threshold: "80"
  - metric: duplicated_lines_density
    operator: GT
    threshold: "3"
  - metric: critical_violations
    operator: GT
    threshold: "0"
  - metric: blocker_violations
    operator: GT
    threshold: "0"
  - metric: security_rating
    operator: GT
    threshold: "1"
```

---

## 6. 部署策略

### 6.1 蓝绿部署

```yaml
# blue-green-deployment.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: dmp-integration-service
spec:
  replicas: 3
  strategy:
    blueGreen:
      activeService: dmp-integration-service-active
      previewService: dmp-integration-service-preview
      autoPromotionEnabled: false
      autoPromotionSeconds: 300
      maxUnavailable: 0
      maxSurge: 100%
```

### 6.2 金丝雀发布

```yaml
# canary-deployment.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: dmp-integration-service
spec:
  replicas: 10
  strategy:
    canary:
      canaryService: dmp-integration-service-canary
      stableService: dmp-integration-service-stable
      trafficRouting:
        nginx:
          stableIngress: dmp-integration-service-ingress
          annotationPrefix: nginx.ingress.kubernetes.io
      steps:
        - setWeight: 10
        - pause: {duration: 10m}
        - setWeight: 25
        - pause: {duration: 10m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 100
```

---

## 7. 监控告警

### 7.1 流水线监控

```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pipeline-alerts
spec:
  groups:
    - name: pipeline
      rules:
        - alert: PipelineFailed
          expr: gitlab_ci_pipeline_status{status="failed"} == 1
          for: 0m
          labels:
            severity: critical
          annotations:
            summary: "Pipeline failed"
            description: "Pipeline {{ $labels.pipeline }} failed"
        
        - alert: PipelineDurationHigh
          expr: gitlab_ci_pipeline_duration_seconds > 1800
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pipeline duration too long"
            description: "Pipeline {{ $labels.pipeline }} took more than 30 minutes"
```

---

**文档结束**
