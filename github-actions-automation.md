# GitHub Actions - CI/CD Automation Platform

## Tổng quan về GitHub Actions

GitHub Actions là nền tảng CI/CD tích hợp sẵn trong GitHub, cho phép tự động hóa workflows trực tiếp từ repository. GitHub Actions sử dụng YAML-based configuration và cung cấp marketplace với hàng nghìn actions có sẵn.

## Workflow Basics

### Workflow Structure
```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 2 * * 1'  # Weekly on Monday at 2 AM
  workflow_dispatch:     # Manual trigger

env:
  NODE_VERSION: '18'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [16, 18, 20]
        os: [ubuntu-latest, windows-latest, macos-latest]
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
        
    - name: Install dependencies
      run: npm ci
      
    - name: Run tests
      run: npm test
      
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage/lcov.info
        
  build:
    needs: test
    runs-on: ubuntu-latest
    
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      
    steps:
    - name: Checkout
      uses: actions/checkout@v4
      
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
      
    - name: Login to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
        
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}
          
    - name: Build and push
      id: build
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        
  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    environment:
      name: production
      url: https://myapp.com
      
    steps:
    - name: Deploy to production
      run: |
        echo "Deploying image with digest: ${{ needs.build.outputs.image-digest }}"
        # Deployment logic here
```

## Advanced Workflows

### Multi-Environment Deployment
```yaml
name: Multi-Environment Deployment

on:
  push:
    branches: [ main, develop, 'release/*' ]

jobs:
  determine-environment:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.env.outputs.environment }}
      deploy: ${{ steps.env.outputs.deploy }}
    steps:
    - name: Determine environment
      id: env
      run: |
        if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
          echo "environment=production" >> $GITHUB_OUTPUT
          echo "deploy=true" >> $GITHUB_OUTPUT
        elif [[ "${{ github.ref }}" == "refs/heads/develop" ]]; then
          echo "environment=staging" >> $GITHUB_OUTPUT
          echo "deploy=true" >> $GITHUB_OUTPUT
        elif [[ "${{ github.ref }}" == refs/heads/release/* ]]; then
          echo "environment=uat" >> $GITHUB_OUTPUT
          echo "deploy=true" >> $GITHUB_OUTPUT
        else
          echo "environment=none" >> $GITHUB_OUTPUT
          echo "deploy=false" >> $GITHUB_OUTPUT
        fi

  build-and-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'
        
    - name: Install and test
      run: |
        npm ci
        npm run lint
        npm run test:unit
        npm run test:integration
        
    - name: Build application
      run: npm run build
      
    - name: Upload build artifacts
      uses: actions/upload-artifact@v4
      with:
        name: build-files
        path: dist/
        retention-days: 30

  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Run Snyk security scan
      uses: snyk/actions/node@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        args: --severity-threshold=high
        
    - name: Run CodeQL analysis
      uses: github/codeql-action/init@v3
      with:
        languages: javascript
        
    - name: Perform CodeQL analysis
      uses: github/codeql-action/analyze@v3

  deploy:
    needs: [determine-environment, build-and-test, security-scan]
    runs-on: ubuntu-latest
    if: needs.determine-environment.outputs.deploy == 'true'
    
    environment:
      name: ${{ needs.determine-environment.outputs.environment }}
      
    steps:
    - uses: actions/checkout@v4
    
    - name: Download build artifacts
      uses: actions/download-artifact@v4
      with:
        name: build-files
        path: dist/
        
    - name: Deploy to ${{ needs.determine-environment.outputs.environment }}
      run: |
        echo "Deploying to ${{ needs.determine-environment.outputs.environment }}"
        # Environment-specific deployment logic
        case "${{ needs.determine-environment.outputs.environment }}" in
          "production")
            echo "Deploying to production cluster"
            kubectl apply -f k8s/production/
            ;;
          "staging")
            echo "Deploying to staging cluster"
            kubectl apply -f k8s/staging/
            ;;
          "uat")
            echo "Deploying to UAT environment"
            kubectl apply -f k8s/uat/
            ;;
        esac
```

### Reusable Workflows
```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deployment Workflow

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image-tag:
        required: true
        type: string
      dry-run:
        required: false
        type: boolean
        default: false
    secrets:
      KUBECONFIG:
        required: true
      SLACK_WEBHOOK:
        required: false
    outputs:
      deployment-url:
        description: "URL of the deployed application"
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    
    outputs:
      url: ${{ steps.deploy.outputs.url }}
      
    steps:
    - name: Checkout
      uses: actions/checkout@v4
      
    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.28.0'
        
    - name: Configure kubeconfig
      run: |
        echo "${{ secrets.KUBECONFIG }}" | base64 -d > kubeconfig
        export KUBECONFIG=kubeconfig
        
    - name: Deploy application
      id: deploy
      run: |
        if [[ "${{ inputs.dry-run }}" == "true" ]]; then
          echo "Dry run mode - would deploy image: ${{ inputs.image-tag }}"
          echo "url=https://dry-run.example.com" >> $GITHUB_OUTPUT
        else
          # Actual deployment
          helm upgrade --install myapp ./helm-chart \
            --set image.tag=${{ inputs.image-tag }} \
            --set environment=${{ inputs.environment }} \
            --namespace ${{ inputs.environment }}
          
          # Get service URL
          URL=$(kubectl get service myapp -n ${{ inputs.environment }} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
          echo "url=https://$URL" >> $GITHUB_OUTPUT
        fi
        
    - name: Notify Slack
      if: always() && secrets.SLACK_WEBHOOK
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
        text: |
          Deployment to ${{ inputs.environment }} ${{ job.status }}
          Image: ${{ inputs.image-tag }}
          URL: ${{ steps.deploy.outputs.url }}

# Using the reusable workflow
# .github/workflows/main.yml
name: Main Pipeline

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
    - name: Build and push image
      # Build logic here
      id: meta
      run: echo "tags=myapp:${{ github.sha }}" >> $GITHUB_OUTPUT

  deploy-staging:
    needs: build
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      image-tag: ${{ needs.build.outputs.image-tag }}
    secrets:
      KUBECONFIG: ${{ secrets.STAGING_KUBECONFIG }}
      SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}

  deploy-production:
    needs: [build, deploy-staging]
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
      image-tag: ${{ needs.build.outputs.image-tag }}
    secrets:
      KUBECONFIG: ${{ secrets.PROD_KUBECONFIG }}
      SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
```

## Custom Actions

### JavaScript Action
```javascript
// action.yml
name: 'Custom Deployment Action'
description: 'Deploy application to Kubernetes'
inputs:
  kubeconfig:
    description: 'Kubernetes config'
    required: true
  namespace:
    description: 'Target namespace'
    required: true
    default: 'default'
  image:
    description: 'Container image'
    required: true
outputs:
  deployment-url:
    description: 'URL of deployed application'
runs:
  using: 'node20'
  main: 'index.js'

// index.js
const core = require('@actions/core');
const exec = require('@actions/exec');
const fs = require('fs');

async function run() {
  try {
    const kubeconfig = core.getInput('kubeconfig');
    const namespace = core.getInput('namespace');
    const image = core.getInput('image');
    
    // Write kubeconfig to file
    fs.writeFileSync('/tmp/kubeconfig', kubeconfig);
    process.env.KUBECONFIG = '/tmp/kubeconfig';
    
    // Deploy to Kubernetes
    await exec.exec('kubectl', [
      'set', 'image', 
      `deployment/myapp`, 
      `myapp=${image}`,
      '-n', namespace
    ]);
    
    // Wait for rollout
    await exec.exec('kubectl', [
      'rollout', 'status',
      'deployment/myapp',
      '-n', namespace
    ]);
    
    // Get service URL
    let serviceUrl = '';
    await exec.exec('kubectl', [
      'get', 'service', 'myapp',
      '-n', namespace,
      '-o', 'jsonpath={.status.loadBalancer.ingress[0].hostname}'
    ], {
      listeners: {
        stdout: (data) => {
          serviceUrl += data.toString();
        }
      }
    });
    
    core.setOutput('deployment-url', `https://${serviceUrl}`);
    core.info(`Deployment successful: https://${serviceUrl}`);
    
  } catch (error) {
    core.setFailed(error.message);
  }
}

run();

// package.json
{
  "name": "custom-deploy-action",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "@actions/core": "^1.10.0",
    "@actions/exec": "^1.1.1"
  }
}
```

### Docker Action
```dockerfile
# Dockerfile
FROM alpine:3.18

RUN apk add --no-cache \
    curl \
    kubectl \
    helm

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]

# entrypoint.sh
#!/bin/sh
set -e

KUBECONFIG_DATA="$1"
NAMESPACE="$2"
IMAGE="$3"

# Setup kubeconfig
echo "$KUBECONFIG_DATA" | base64 -d > /tmp/kubeconfig
export KUBECONFIG=/tmp/kubeconfig

# Deploy application
helm upgrade --install myapp ./helm-chart \
  --set image.repository="${IMAGE%:*}" \
  --set image.tag="${IMAGE##*:}" \
  --namespace "$NAMESPACE" \
  --create-namespace

# Wait for deployment
kubectl rollout status deployment/myapp -n "$NAMESPACE"

# Get service URL
SERVICE_URL=$(kubectl get service myapp -n "$NAMESPACE" -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo "deployment-url=https://$SERVICE_URL" >> $GITHUB_OUTPUT

# action.yml
name: 'Docker Deploy Action'
description: 'Deploy using Docker container'
inputs:
  kubeconfig:
    description: 'Base64 encoded kubeconfig'
    required: true
  namespace:
    description: 'Target namespace'
    required: true
  image:
    description: 'Container image'
    required: true
outputs:
  deployment-url:
    description: 'Deployment URL'
runs:
  using: 'docker'
  image: 'Dockerfile'
  args:
    - ${{ inputs.kubeconfig }}
    - ${{ inputs.namespace }}
    - ${{ inputs.image }}
```

## Matrix Strategies

### Complex Matrix Builds
```yaml
name: Matrix Testing

on: [push, pull_request]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [16, 18, 20]
        include:
          # Additional configurations
          - os: ubuntu-latest
            node-version: 18
            coverage: true
          - os: windows-latest
            node-version: 20
            experimental: true
        exclude:
          # Skip problematic combinations
          - os: macos-latest
            node-version: 16
            
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node-version }}
        
    - name: Install dependencies
      run: npm ci
      
    - name: Run tests
      run: npm test
      
    - name: Run coverage
      if: matrix.coverage
      run: |
        npm run test:coverage
        npx codecov
        
    - name: Run experimental tests
      if: matrix.experimental
      continue-on-error: true
      run: npm run test:experimental

  build-matrix:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        platform: [linux/amd64, linux/arm64, linux/arm/v7]
        
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up QEMU
      uses: docker/setup-qemu-action@v3
      
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
      
    - name: Build for ${{ matrix.platform }}
      uses: docker/build-push-action@v5
      with:
        context: .
        platforms: ${{ matrix.platform }}
        tags: myapp:${{ matrix.platform }}
        outputs: type=image,push=false
```

## Secrets Management

### Environment-specific Secrets
```yaml
name: Secure Deployment

on:
  push:
    branches: [ main, develop ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        environment: [staging, production]
        
    environment: ${{ matrix.environment }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ vars.AWS_REGION }}
        
    - name: Deploy to ${{ matrix.environment }}
      env:
        DATABASE_URL: ${{ secrets.DATABASE_URL }}
        API_KEY: ${{ secrets.API_KEY }}
        ENVIRONMENT: ${{ matrix.environment }}
      run: |
        echo "Deploying to $ENVIRONMENT"
        # Use environment variables securely
        ./deploy.sh --env=$ENVIRONMENT --db-url="$DATABASE_URL"

  vault-integration:
    runs-on: ubuntu-latest
    steps:
    - name: Import secrets from Vault
      uses: hashicorp/vault-action@v2
      with:
        url: ${{ secrets.VAULT_URL }}
        token: ${{ secrets.VAULT_TOKEN }}
        secrets: |
          secret/data/myapp database_password | DATABASE_PASSWORD ;
          secret/data/myapp api_key | API_KEY
          
    - name: Use secrets
      run: |
        echo "Database password length: ${#DATABASE_PASSWORD}"
        echo "API key starts with: ${API_KEY:0:8}..."
```

## Monitoring và Observability

### Workflow Monitoring
```yaml
name: Monitored Pipeline

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Start build metrics
      run: |
        curl -X POST "${{ secrets.METRICS_ENDPOINT }}/start" \
          -H "Content-Type: application/json" \
          -d '{
            "workflow": "${{ github.workflow }}",
            "job": "${{ github.job }}",
            "run_id": "${{ github.run_id }}",
            "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"
          }'
          
    - name: Build application
      id: build
      run: |
        start_time=$(date +%s)
        npm ci
        npm run build
        end_time=$(date +%s)
        echo "duration=$((end_time - start_time))" >> $GITHUB_OUTPUT
        
    - name: Send build metrics
      if: always()
      run: |
        curl -X POST "${{ secrets.METRICS_ENDPOINT }}/build" \
          -H "Content-Type: application/json" \
          -d '{
            "workflow": "${{ github.workflow }}",
            "job": "${{ github.job }}",
            "run_id": "${{ github.run_id }}",
            "status": "${{ job.status }}",
            "duration": ${{ steps.build.outputs.duration }},
            "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"
          }'
          
    - name: Upload build logs
      if: failure()
      uses: actions/upload-artifact@v4
      with:
        name: build-logs
        path: |
          npm-debug.log*
          yarn-error.log*
          
  notify:
    needs: build
    runs-on: ubuntu-latest
    if: always()
    
    steps:
    - name: Notify Slack
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ needs.build.result }}
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
        channel: '#ci-cd'
        username: 'GitHub Actions'
        icon_emoji: ':github:'
        text: |
          Workflow: ${{ github.workflow }}
          Repository: ${{ github.repository }}
          Branch: ${{ github.ref_name }}
          Commit: ${{ github.sha }}
          Status: ${{ needs.build.result }}
          
    - name: Create GitHub issue on failure
      if: needs.build.result == 'failure' && github.ref == 'refs/heads/main'
      uses: actions/github-script@v7
      with:
        script: |
          github.rest.issues.create({
            owner: context.repo.owner,
            repo: context.repo.repo,
            title: `Build failure on main branch`,
            body: `
              **Workflow:** ${{ github.workflow }}
              **Run ID:** ${{ github.run_id }}
              **Commit:** ${{ github.sha }}
              **Author:** ${{ github.actor }}
              
              The build failed on the main branch. Please investigate and fix the issue.
              
              [View workflow run](https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }})
            `,
            labels: ['bug', 'ci-failure']
          })
```

## Performance Optimization

### Caching Strategies
```yaml
name: Optimized Pipeline

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    # Cache dependencies
    - name: Cache Node modules
      uses: actions/cache@v3
      with:
        path: ~/.npm
        key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          ${{ runner.os }}-node-
          
    # Cache build outputs
    - name: Cache build
      uses: actions/cache@v3
      with:
        path: |
          dist/
          .next/cache/
        key: ${{ runner.os }}-build-${{ github.sha }}
        restore-keys: |
          ${{ runner.os }}-build-
          
    # Cache Docker layers
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
      
    - name: Build with cache
      uses: docker/build-push-action@v5
      with:
        context: .
        cache-from: type=gha
        cache-to: type=gha,mode=max
        
  parallel-jobs:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    # Run jobs in parallel using job matrix
    - name: Parallel testing
      run: |
        # Split tests into chunks
        npm run test:unit -- --shard=1/4 &
        npm run test:unit -- --shard=2/4 &
        npm run test:unit -- --shard=3/4 &
        npm run test:unit -- --shard=4/4 &
        wait
```

GitHub Actions cung cấp một nền tảng CI/CD mạnh mẽ và linh hoạt với tích hợp sâu vào GitHub ecosystem, phù hợp cho mọi loại dự án từ nhỏ đến lớn.