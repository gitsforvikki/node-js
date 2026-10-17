# Lesson 217 — Jenkins Pipeline Concepts

## 1. What is Jenkins?
Jenkins is an automation server. Its **controller** coordinates jobs and **agents** run build steps. A `Jenkinsfile` describes the pipeline as code.

```text
GitHub -> webhook -> Jenkins Controller -> Agent
                                     -> Checkout
                                     -> Install
                                     -> Test
                                     -> Build
                                     -> Deploy (approved)
```

## 2. Declarative Jenkinsfile
```groovy
pipeline {
  agent any
  tools { nodejs 'Node22' }

  options {
    timestamps()
    timeout(time: 20, unit: 'MINUTES')
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Install') {
      steps { sh 'npm ci' }
    }
    stage('Test') {
      steps { sh 'npm test' }
    }
    stage('Build') {
      steps { sh 'npm run build' }
    }
  }

  post {
    success { echo 'Tests and build passed' }
    failure { echo 'Pipeline failed' }
  }
}
```

**Prerequisites:** a configured agent, Jenkins NodeJS plugin with a tool named `Node22`, and an existing build script. For plain JS without build, remove that stage.

## 3. Execution and failures
A stage runs only after required earlier stages succeed. If `npm test` exits nonzero, the pipeline fails and deployment must not proceed. Jobs should publish test reports, log failures and use timeouts to avoid hanging forever.

## 4. Jenkins credentials
**Never write secrets directly in the Jenkinsfile.** Use the credentials store and scoped injection.

```groovy
withCredentials([string(credentialsId: 'registry-token', variable: 'REGISTRY_TOKEN')]) {
  sh '''
    set +x
    printf '%s' "$REGISTRY_TOKEN" | docker login registry.example \
      --username ci-user --password-stdin
  '''
}
```

Credentials Binding plugin and a trusted agent are required. Masking is not a complete security boundary: untrusted pipeline code can leak credentials. Restrict deployment pipeline changes, agents and permissions.

## 5. Image publishing and deployment
Typical stages: build versioned Docker image → run image scan → push registry → staging deployment → health check → approval → production deployment. Docker access on Jenkins agents is powerful and must be isolated.

## 6. Pipeline terminology
- **Job:** configured work.
- **Pipeline:** full workflow.
- **Stage:** named phase.
- **Step:** one command/action.
- **Agent:** environment running steps.

## 7. Common mistakes
Running untrusted PR code with production credentials; installing with `npm install` instead of `npm ci`; building on the controller; no rollback; no timeout; printing tokens into logs.

## Practice
Implement Checkout → Install → Test → Build; break a test and verify Build is skipped. Then add a staging step triggered only from a protected branch.

**Interview answer:** Jenkins automates CI/CD with version-controlled pipelines, named stages and build agents. Production use requires secure credential handling, isolated agents, tested releases and protected deployment approvals.
