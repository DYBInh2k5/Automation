# Jenkins - CI/CD Automation Platform

## Tổng quan về Jenkins

Jenkins là nền tảng automation mã nguồn mở hàng đầu cho Continuous Integration và Continuous Deployment (CI/CD). Jenkins hỗ trợ building, testing và deploying code tự động thông qua pipeline workflows.

## Cài đặt và Cấu hình

### Docker Installation
```bash
# Pull Jenkins image
docker pull jenkins/jenkins:lts

# Run Jenkins container
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts

# Get initial admin password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### Docker Compose Setup
```yaml
version: '3.8'
services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - JENKINS_OPTS="--httpPort=8080"
    restart: unless-stopped

  jenkins-agent:
    image: jenkins/ssh-agent:latest
    container_name: jenkins-agent
    environment:
      - JENKINS_AGENT_SSH_PUBKEY=ssh-rsa AAAAB3NzaC1yc2E...

volumes:
  jenkins_home:
```

### Kubernetes Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        ports:
        - containerPort: 8080
        - containerPort: 50000
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
        env:
        - name: JAVA_OPTS
          value: "-Xmx2048m"
      volumes:
      - name: jenkins-home
        persistentVolumeClaim:
          claimName: jenkins-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  namespace: jenkins
spec:
  selector:
    app: jenkins
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: agent
    port: 50000
    targetPort: 50000
  type: LoadBalancer
```

## Pipeline Development

### Declarative Pipeline
```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'your-registry.com'
        IMAGE_NAME = 'myapp'
        KUBECONFIG = credentials('kubeconfig')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                }
            }
        }
        
        stage('Build') {
            steps {
                sh '''
                    echo "Building application..."
                    npm install
                    npm run build
                '''
            }
        }
        
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm run test:unit'
                    }
                    post {
                        always {
                            publishTestResults testResultsPattern: 'test-results.xml'
                        }
                    }
                }
                
                stage('Integration Tests') {
                    steps {
                        sh 'npm run test:integration'
                    }
                }
                
                stage('Security Scan') {
                    steps {
                        sh 'npm audit --audit-level moderate'
                    }
                }
            }
        }
        
        stage('Code Quality') {
            steps {
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=myapp \
                            -Dsonar.sources=src \
                            -Dsonar.tests=tests \
                            -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
                        """
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    def image = docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}:${env.GIT_COMMIT_SHORT}")
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-registry-credentials') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                script {
                    kubernetesDeploy(
                        configs: 'k8s/staging/*.yaml',
                        kubeconfigId: 'kubeconfig',
                        enableConfigSubstitution: true
                    )
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                script {
                    kubernetesDeploy(
                        configs: 'k8s/production/*.yaml',
                        kubeconfigId: 'kubeconfig',
                        enableConfigSubstitution: true
                    )
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: "✅ Pipeline succeeded for ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
            )
        }
        failure {
            slackSend(
                channel: '#deployments',
                color: 'danger',
                message: "❌ Pipeline failed for ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
            )
        }
    }
}
```

### Scripted Pipeline
```groovy
node {
    def app
    def imageTag = "${env.BUILD_NUMBER}"
    
    try {
        stage('Checkout') {
            checkout scm
        }
        
        stage('Build Application') {
            sh '''
                echo "Installing dependencies..."
                npm ci
                echo "Building application..."
                npm run build
            '''
        }
        
        stage('Run Tests') {
            parallel(
                "Unit Tests": {
                    sh 'npm run test:unit'
                    publishTestResults testResultsPattern: 'junit.xml'
                },
                "Lint": {
                    sh 'npm run lint'
                    publishCheckStyleResults pattern: 'eslint-report.xml'
                },
                "Security Audit": {
                    sh 'npm audit --json > audit-results.json'
                    archiveArtifacts artifacts: 'audit-results.json'
                }
            )
        }
        
        stage('Build Docker Image') {
            app = docker.build("myapp:${imageTag}")
        }
        
        stage('Push to Registry') {
            docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                app.push("${imageTag}")
                app.push("latest")
            }
        }
        
        stage('Deploy to Staging') {
            sh """
                helm upgrade --install myapp-staging ./helm-chart \
                --set image.tag=${imageTag} \
                --set environment=staging \
                --namespace staging
            """
        }
        
        stage('Integration Tests') {
            sh '''
                sleep 30  # Wait for deployment
                npm run test:e2e -- --env=staging
            '''
        }
        
        stage('Deploy to Production') {
            input message: 'Approve deployment to production?'
            sh """
                helm upgrade --install myapp-prod ./helm-chart \
                --set image.tag=${imageTag} \
                --set environment=production \
                --namespace production
            """
        }
        
    } catch (Exception e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        // Cleanup
        sh 'docker system prune -f'
    }
}
```

## Multi-branch Pipeline
```groovy
// Jenkinsfile for multi-branch pipeline
pipeline {
    agent any
    
    parameters {
        choice(
            name: 'DEPLOY_ENV',
            choices: ['dev', 'staging', 'prod'],
            description: 'Target deployment environment'
        )
        booleanParam(
            name: 'SKIP_TESTS',
            defaultValue: false,
            description: 'Skip test execution'
        )
    }
    
    stages {
        stage('Setup') {
            steps {
                script {
                    // Set environment-specific variables
                    switch(env.BRANCH_NAME) {
                        case 'main':
                            env.DEPLOY_ENV = 'prod'
                            env.IMAGE_TAG = 'stable'
                            break
                        case 'develop':
                            env.DEPLOY_ENV = 'staging'
                            env.IMAGE_TAG = 'latest'
                            break
                        default:
                            env.DEPLOY_ENV = 'dev'
                            env.IMAGE_TAG = env.BRANCH_NAME.replaceAll('/', '-')
                    }
                }
            }
        }
        
        stage('Build & Test') {
            when {
                not { params.SKIP_TESTS }
            }
            matrix {
                axes {
                    axis {
                        name 'NODE_VERSION'
                        values '14', '16', '18'
                    }
                }
                stages {
                    stage('Test') {
                        steps {
                            sh """
                                nvm use ${NODE_VERSION}
                                npm ci
                                npm test
                            """
                        }
                    }
                }
            }
        }
        
        stage('Deploy') {
            steps {
                script {
                    deployToEnvironment(env.DEPLOY_ENV, env.IMAGE_TAG)
                }
            }
        }
    }
}

def deployToEnvironment(environment, imageTag) {
    echo "Deploying to ${environment} with tag ${imageTag}"
    
    switch(environment) {
        case 'dev':
            sh "kubectl apply -f k8s/dev/ --namespace=dev"
            break
        case 'staging':
            sh "kubectl apply -f k8s/staging/ --namespace=staging"
            break
        case 'prod':
            input message: 'Approve production deployment?'
            sh "kubectl apply -f k8s/prod/ --namespace=prod"
            break
    }
}
```

## Plugin Configuration

### Essential Plugins
```groovy
// plugins.txt for automated installation
ant:latest
build-timeout:latest
credentials-binding:latest
timestamper:latest
ws-cleanup:latest
github-branch-source:latest
pipeline-github-lib:latest
pipeline-stage-view:latest
git:latest
ssh-slaves:latest
matrix-auth:latest
pam-auth:latest
ldap:latest
email-ext:latest
mailer:latest
slack:latest
docker-workflow:latest
kubernetes:latest
helm:latest
sonar:latest
```

### Plugin Configuration as Code
```yaml
# jenkins.yaml for Configuration as Code plugin
jenkins:
  systemMessage: "Jenkins configured automatically by JCasC"
  
  securityRealm:
    ldap:
      configurations:
        - server: "ldap://ldap.company.com:389"
          rootDN: "dc=company,dc=com"
          userSearchBase: "ou=users"
          userSearch: "uid={0}"
          groupSearchBase: "ou=groups"
          
  authorizationStrategy:
    globalMatrix:
      permissions:
        - "Overall/Administer:admin"
        - "Overall/Read:authenticated"
        - "Job/Build:developers"
        - "Job/Read:developers"

  nodes:
    - permanent:
        name: "linux-agent"
        remoteFS: "/home/jenkins"
        launcher:
          ssh:
            host: "agent.company.com"
            credentialsId: "ssh-key"
            
  clouds:
    - kubernetes:
        name: "kubernetes"
        serverUrl: "https://kubernetes.company.com"
        namespace: "jenkins"
        credentialsId: "kubeconfig"
        templates:
          - name: "maven-agent"
            label: "maven"
            containers:
              - name: "maven"
                image: "maven:3.8-openjdk-11"
                command: "sleep"
                args: "99d"

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "github-credentials"
              username: "jenkins-bot"
              password: "${GITHUB_TOKEN}"
          - string:
              scope: GLOBAL
              id: "slack-token"
              secret: "${SLACK_TOKEN}"
          - file:
              scope: GLOBAL
              id: "kubeconfig"
              fileName: "kubeconfig"
              secretBytes: "${KUBECONFIG_BASE64}"

tool:
  git:
    installations:
      - name: "Default"
        home: "/usr/bin/git"
  maven:
    installations:
      - name: "Maven-3.8"
        properties:
          - installSource:
              installers:
                - maven:
                    id: "3.8.6"
  nodejs:
    installations:
      - name: "NodeJS-16"
        properties:
          - installSource:
              installers:
                - nodeJSInstaller:
                    id: "16.17.0"
```

## Shared Libraries

### Library Structure
```
vars/
├── buildDockerImage.groovy
├── deployToKubernetes.groovy
├── runTests.groovy
└── sendNotification.groovy

src/
└── com/
    └── company/
        └── jenkins/
            ├── Docker.groovy
            ├── Kubernetes.groovy
            └── Notifications.groovy

resources/
├── scripts/
│   ├── deploy.sh
│   └── test.sh
└── templates/
    ├── Dockerfile.template
    └── deployment.yaml.template
```

### Shared Library Functions
```groovy
// vars/buildDockerImage.groovy
def call(Map config) {
    def imageName = config.imageName ?: env.JOB_NAME.toLowerCase()
    def imageTag = config.imageTag ?: env.BUILD_NUMBER
    def dockerfile = config.dockerfile ?: 'Dockerfile'
    def context = config.context ?: '.'
    
    script {
        def image = docker.build("${imageName}:${imageTag}", "-f ${dockerfile} ${context}")
        
        if (config.push) {
            docker.withRegistry(config.registry, config.credentials) {
                image.push()
                if (config.pushLatest) {
                    image.push('latest')
                }
            }
        }
        
        return image
    }
}

// vars/deployToKubernetes.groovy
def call(Map config) {
    def namespace = config.namespace ?: 'default'
    def manifests = config.manifests ?: 'k8s/'
    def kubeconfig = config.kubeconfig ?: 'kubeconfig'
    
    withCredentials([file(credentialsId: kubeconfig, variable: 'KUBECONFIG')]) {
        sh """
            kubectl apply -f ${manifests} --namespace=${namespace}
            kubectl rollout status deployment/${config.deployment} --namespace=${namespace}
        """
    }
}

// vars/runTests.groovy
def call(Map config) {
    def testType = config.type ?: 'unit'
    def testCommand = config.command ?: 'npm test'
    def publishResults = config.publishResults ?: true
    
    try {
        sh testCommand
    } finally {
        if (publishResults && config.resultsPattern) {
            publishTestResults testResultsPattern: config.resultsPattern
        }
        if (config.coveragePattern) {
            publishCoverage adapters: [
                coberturaAdapter(config.coveragePattern)
            ]
        }
    }
}

// src/com/company/jenkins/Docker.groovy
package com.company.jenkins

class Docker implements Serializable {
    def script
    
    Docker(script) {
        this.script = script
    }
    
    def buildAndPush(imageName, imageTag, registry, credentials) {
        script.echo "Building Docker image: ${imageName}:${imageTag}"
        
        def image = script.docker.build("${imageName}:${imageTag}")
        
        script.docker.withRegistry(registry, credentials) {
            image.push()
            image.push('latest')
        }
        
        return image
    }
    
    def scanImage(imageName, imageTag) {
        script.sh """
            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy image ${imageName}:${imageTag}
        """
    }
}
```

### Using Shared Libraries
```groovy
// Jenkinsfile using shared library
@Library('jenkins-shared-library@main') _

pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                buildDockerImage([
                    imageName: 'myapp',
                    imageTag: env.BUILD_NUMBER,
                    push: true,
                    registry: 'https://registry.company.com',
                    credentials: 'docker-registry'
                ])
            }
        }
        
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        runTests([
                            type: 'unit',
                            command: 'npm run test:unit',
                            resultsPattern: 'test-results.xml',
                            coveragePattern: 'coverage/cobertura.xml'
                        ])
                    }
                }
                
                stage('Integration Tests') {
                    steps {
                        runTests([
                            type: 'integration',
                            command: 'npm run test:integration'
                        ])
                    }
                }
            }
        }
        
        stage('Deploy') {
            steps {
                deployToKubernetes([
                    namespace: 'production',
                    manifests: 'k8s/production/',
                    deployment: 'myapp',
                    kubeconfig: 'prod-kubeconfig'
                ])
            }
        }
    }
    
    post {
        always {
            sendNotification([
                channel: '#deployments',
                status: currentBuild.result
            ])
        }
    }
}
```

## Security Configuration

### Security Best Practices
```groovy
// Security configuration
jenkins:
  security:
    globalJobDslSecurityConfiguration:
      useScriptSecurity: true
    
  crumbIssuer:
    standard:
      excludeClientIPFromCrumb: false
      
  remotingSecurity:
    enabled: true

security:
  scriptApproval:
    approvedSignatures:
      - "method java.lang.String toLowerCase"
      - "method java.util.Collection size"
      
  globalMatrix:
    permissions:
      - "Overall/Administer:admin"
      - "Overall/Read:authenticated"
      - "Job/Build:developers"
      - "Job/Configure:developers"
      - "Job/Read:developers"
      - "Job/Workspace:developers"
```

### Credential Management
```groovy
// Managing credentials in pipeline
pipeline {
    agent any
    
    environment {
        // Using credential binding
        DOCKER_CREDS = credentials('docker-registry')
        API_KEY = credentials('api-key')
    }
    
    stages {
        stage('Deploy') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'database-creds',
                        usernameVariable: 'DB_USER',
                        passwordVariable: 'DB_PASS'
                    ),
                    string(
                        credentialsId: 'secret-token',
                        variable: 'SECRET_TOKEN'
                    ),
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        echo "Deploying with user: $DB_USER"
                        kubectl --kubeconfig=$KUBECONFIG_FILE apply -f deployment.yaml
                    '''
                }
            }
        }
    }
}
```

## Monitoring và Alerting

### Prometheus Integration
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'jenkins'
    static_configs:
      - targets: ['jenkins:8080']
    metrics_path: '/prometheus'
    scrape_interval: 30s
```

### Grafana Dashboard
```json
{
  "dashboard": {
    "title": "Jenkins CI/CD Metrics",
    "panels": [
      {
        "title": "Build Success Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(jenkins_builds_success_total[5m]) / rate(jenkins_builds_total[5m]) * 100"
          }
        ]
      },
      {
        "title": "Build Duration",
        "type": "graph",
        "targets": [
          {
            "expr": "jenkins_build_duration_milliseconds"
          }
        ]
      },
      {
        "title": "Queue Length",
        "type": "graph", 
        "targets": [
          {
            "expr": "jenkins_queue_size_value"
          }
        ]
      }
    ]
  }
}
```

### Custom Monitoring
```groovy
// Pipeline with custom metrics
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                script {
                    def startTime = System.currentTimeMillis()
                    
                    // Build logic here
                    sh 'npm run build'
                    
                    def duration = System.currentTimeMillis() - startTime
                    
                    // Send custom metric
                    sh """
                        curl -X POST http://pushgateway:9091/metrics/job/jenkins/instance/${env.JOB_NAME} \
                        --data-binary 'build_duration_seconds ${duration/1000}'
                    """
                }
            }
        }
    }
}
```

Jenkins cung cấp một nền tảng CI/CD mạnh mẽ và linh hoạt với khả năng mở rộng cao thông qua plugins và shared libraries, phù hợp cho mọi quy mô tổ chức.