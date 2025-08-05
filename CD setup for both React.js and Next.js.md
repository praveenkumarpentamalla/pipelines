Sure! Here's a **complete Jenkins CI/CD setup** for both **React.js** and **Next.js** projects using:

* ✅ Unit Tests
* ✅ Integration/Component Tests
* ✅ Linting (ESLint)
* ✅ Security Scans (npm audit)
* ✅ Code Coverage
* ✅ Build Verification Tests (Smoke Tests)
* ✅ E2E/UI Tests (using Playwright or Cypress)
* ✅ Docker Image Build
* ✅ Docker Scout Scan
* ✅ DockerHub Push

---

## 🔧 Pre-requisites for Jenkins Setup

Install the following on your Jenkins agent:

* `Node.js` (v18+)
* `npm` or `yarn`
* `Docker`
* Jenkins Credentials:

  * `dockerhub-username` (Secret Text)
  * `dockerhub-password` (Secret Text)

Recommended project structure:

```
my-app/
├── package.json
├── .eslintrc
├── jest.config.js
├── cypress/
│   └── e2e/
├── tests/
│   ├── unit/
│   └── integration/
└── Dockerfile
```

---

# 📄 1. Jenkins **Pipeline Job** (React or Next.js)

```groovy
pipeline {
    agent any

    environment {
        DOCKERHUB_USER = credentials('dockerhub-username')
        DOCKERHUB_PASS = credentials('dockerhub-password')
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    npm install
                '''
            }
        }

        stage('Linting - ESLint') {
            steps {
                sh 'npx eslint . || true'
            }
        }

        stage('Security Scan - npm audit') {
            steps {
                sh 'npm audit --audit-level=high || true'
            }
        }

        stage('Unit & Integration Tests + Coverage') {
            steps {
                sh '''
                    npx jest --coverage --runTestsByPath tests/unit/ tests/integration/
                '''
            }
        }

        stage('Smoke Tests') {
            steps {
                sh '''
                    npm run build
                    npm run start &

                    # Add a sleep to wait for server to start
                    sleep 10

                    curl -f http://localhost:3000 || exit 1
                '''
            }
        }

        stage('E2E/UI Tests (Playwright or Cypress)') {
            steps {
                sh '''
                    npx playwright install || true
                    npx playwright test || true
                    # Or for Cypress:
                    # npx cypress run
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t my-next-react-app:latest .
                '''
            }
        }

        stage('Docker Scout Scan') {
            steps {
                sh '''
                    curl -fsSL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh -o install-scout.sh
                    sh install-scout.sh

                    docker scout quickview my-next-react-app:latest || true
                    docker scout cves my-next-react-app:latest --only-severities critical,high --output sarif > scout-report.sarif || true
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                    echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin
                    docker tag my-next-react-app:latest $DOCKERHUB_USER/my-next-react-app:latest
                    docker push $DOCKERHUB_USER/my-next-react-app:latest
                '''
            }
        }

        stage('Archive Reports') {
            steps {
                archiveArtifacts artifacts: 'coverage/**, scout-report.sarif', fingerprint: true
            }
        }

    }
}
```

---

# 🛠 2. Jenkins **Freestyle Job** (React or Next.js)

### ➕ Configure:

* **Source Code Management** → Git (your repo)
* **Build Environment**:

  * Secret texts:

    * `DOCKERHUB_USERNAME`
    * `DOCKERHUB_PASSWORD`
* **Build Step** → Execute Shell:

```bash
#!/bin/bash

echo "Installing dependencies..."
npm install

echo "Running ESLint..."
npx eslint . || true

echo "Running Security Scan (npm audit)..."
npm audit --audit-level=high || true

echo "Running Unit + Integration Tests..."
npx jest --coverage --runTestsByPath tests/unit/ tests/integration/

echo "Running Smoke Tests..."
npm run build
npm run start &
sleep 10
curl -f http://localhost:3000 || exit 1

echo "Running E2E/UI Tests..."
npx playwright install || true
npx playwright test || true
# or: npx cypress run

echo "Building Docker image..."
docker build -t my-next-react-app:latest .

echo "Installing Docker Scout..."
curl -fsSL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh -o install-scout.sh
sh install-scout.sh

echo "Running Docker Scout Scan..."
docker scout quickview my-next-react-app:latest || true
docker scout cves my-next-react-app:latest --only-severities critical,high --output sarif > scout-report.sarif || true

echo "Pushing Docker image..."
echo "$DOCKERHUB_PASSWORD" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin
docker tag my-next-react-app:latest $DOCKERHUB_USERNAME/my-next-react-app:latest
docker push $DOCKERHUB_USERNAME/my-next-react-app:latest
```

* **Post-build Action** → Archive artifacts:

  ```
  coverage/**, scout-report.sarif
  ```

---

## 🚀 Summary Comparison

| Feature                        | Pipeline Job | Freestyle Job |
| ------------------------------ | ------------ | ------------- |
| Git Checkout                   | ✅            | ✅             |
| Lint (ESLint)                  | ✅            | ✅             |
| npm audit                      | ✅            | ✅             |
| Jest Tests + Coverage          | ✅            | ✅             |
| Smoke Tests                    | ✅            | ✅             |
| E2E Tests (Playwright/Cypress) | ✅            | ✅             |
| Docker Image Build + Push      | ✅            | ✅             |
| Docker Scout Scan              | ✅            | ✅             |
| Report Archiving               | ✅            | ✅             |

---

## 🧪 Extra Recommendations

* Add GitHub webhook for auto-trigger
* Use `dotenv` for env configs in Node apps
* Use `docker-compose` if you have frontend + backend
* Split prod/dev Docker builds for optimization

---
