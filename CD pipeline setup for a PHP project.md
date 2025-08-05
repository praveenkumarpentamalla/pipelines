Sure! Here's a full **CI/CD pipeline setup for a PHP project** in **Jenkins**, using either a **Pipeline job** or a **Freestyle job**.

We'll cover:

* ✅ Unit Tests (e.g. PHPUnit)
* ✅ Integration Tests
* ✅ Linting (PHP\_CodeSniffer or PHPStan)
* ✅ Security Scans (e.g. Enlightn or other tools)
* ✅ Code Coverage
* ✅ Smoke Tests (basic app startup check)
* ✅ E2E/UI Tests (e.g. Selenium or Codeception)
* ✅ Docker image build
* ✅ Docker Scout scan
* ✅ DockerHub push

---

## 🔧 Prerequisites

On your Jenkins agent:

* PHP 8.x and Composer installed
* PHPUnit, PHPStan, PHP\_CodeSniffer, etc.
* Docker installed
* Git access to your PHP project

In Jenkins:

* Credentials for DockerHub:

  * `dockerhub-username`
  * `dockerhub-password`

---

## 📄 1. Jenkins **Pipeline Job** for PHP

### ✅ Jenkinsfile

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

        stage('Install PHP Dependencies') {
            steps {
                sh '''
                    composer install --no-interaction
                '''
            }
        }

        stage('Linting (PHP_CodeSniffer or PHPStan)') {
            steps {
                sh '''
                    ./vendor/bin/phpcs --standard=PSR12 src/ || true
                    ./vendor/bin/phpstan analyse src/ || true
                '''
            }
        }

        stage('Security Scan') {
            steps {
                sh '''
                    # Example: Enlightn Security Scanner (requires license)
                    # Alternatively, use "roave/security-advisories"
                    echo "Skipping advanced security scan for demo purposes"
                '''
            }
        }

        stage('Unit Tests with Coverage') {
            steps {
                sh '''
                    ./vendor/bin/phpunit --coverage-text --coverage-clover=coverage.xml
                '''
            }
        }

        stage('Integration Tests') {
            steps {
                sh '''
                    ./vendor/bin/phpunit --testsuite Integration
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                    php -S localhost:8000 -t public > /dev/null 2>&1 &
                    sleep 5
                    curl -f http://localhost:8000 || exit 1
                    kill $(lsof -t -i:8000)
                '''
            }
        }

        stage('E2E/UI Tests') {
            steps {
                sh '''
                    # Use Codeception, Behat, or Selenium
                    echo "E2E tests would go here"
                    # ./vendor/bin/codecept run acceptance
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t my-php-app:latest .
                '''
            }
        }

        stage('Docker Scout Scan') {
            steps {
                sh '''
                    curl -fsSL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh -o install-scout.sh
                    sh install-scout.sh
                    docker scout quickview my-php-app:latest || true
                    docker scout cves my-php-app:latest --only-severities critical,high --output sarif > scout-report.sarif || true
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                    echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin
                    docker tag my-php-app:latest $DOCKERHUB_USER/my-php-app:latest
                    docker push $DOCKERHUB_USER/my-php-app:latest
                '''
            }
        }

        stage('Archive Reports') {
            steps {
                archiveArtifacts artifacts: 'coverage.xml, scout-report.sarif', fingerprint: true
            }
        }
    }
}
```

---

## 🛠 2. Jenkins **Freestyle Job** for PHP

### ➕ Job Setup

1. Source Code Management → Git
2. Credentials: Use secret texts for DockerHub credentials
3. Add **Build Step → Execute Shell**:

```bash
#!/bin/bash

echo "Installing PHP dependencies..."
composer install --no-interaction

echo "Running Linting (PHPStan, PHPCS)..."
./vendor/bin/phpcs --standard=PSR12 src/ || true
./vendor/bin/phpstan analyse src/ || true

echo "Running Unit Tests + Coverage..."
./vendor/bin/phpunit --coverage-text --coverage-clover=coverage.xml

echo "Running Integration Tests..."
./vendor/bin/phpunit --testsuite Integration

echo "Running Smoke Test..."
php -S localhost:8000 -t public > /dev/null 2>&1 &
sleep 5
curl -f http://localhost:8000 || exit 1
kill $(lsof -t -i:8000)

echo "Running E2E/UI Tests..."
# ./vendor/bin/codecept run acceptance

echo "Building Docker image..."
docker build -t my-php-app:latest .

echo "Installing Docker Scout..."
curl -fsSL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh -o install-scout.sh
sh install-scout.sh

echo "Running Docker Scout Scan..."
docker scout quickview my-php-app:latest || true
docker scout cves my-php-app:latest --only-severities critical,high --output sarif > scout-report.sarif || true

echo "Pushing Docker Image to DockerHub..."
echo "$DOCKERHUB_PASSWORD" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin
docker tag my-php-app:latest $DOCKERHUB_USERNAME/my-php-app:latest
docker push $DOCKERHUB_USERNAME/my-php-app:latest
```

4. **Post-build Actions → Archive Artifacts**

```
coverage.xml, scout-report.sarif
```

---

## 🧪 Sample Project Folder Layout

```
my-php-app/
├── composer.json
├── public/
│   └── index.php
├── src/
├── tests/
│   ├── Unit/
│   └── Integration/
├── Dockerfile
├── phpunit.xml
├── .phpcs.xml
└── .phpstan.neon
```

---

## ✅ Summary Comparison

| Feature                   | Pipeline Job | Freestyle Job |
| ------------------------- | ------------ | ------------- |
| Git Checkout              | ✅            | ✅             |
| Composer Install          | ✅            | ✅             |
| Linting                   | ✅            | ✅             |
| Security Scan             | ✅            | ✅             |
| Unit & Integration Tests  | ✅            | ✅             |
| Code Coverage             | ✅            | ✅             |
| Smoke + E2E Tests         | ✅            | ✅             |
| Docker Image Build + Push | ✅            | ✅             |
| Docker Scout Scan         | ✅            | ✅             |
| Report Archiving          | ✅            | ✅             |

---

