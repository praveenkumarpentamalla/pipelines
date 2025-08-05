Here’s a complete **CI/CD pipeline for a Laravel project** using **Jenkins**, both as a:

* ✅ **Pipeline job (Jenkinsfile)**
* ✅ **Freestyle job (Shell script)**

It includes:

* ✅ Code checkout
* ✅ Composer dependency install
* ✅ Linting (PHPStan, PHP\_CodeSniffer)
* ✅ Security scans (roave/security-advisories)
* ✅ Unit & integration tests with PHPUnit
* ✅ Code coverage
* ✅ Smoke tests
* ✅ E2E/UI tests (Laravel Dusk or Codeception)
* ✅ Docker image build
* ✅ Docker image scan (Docker Scout)
* ✅ DockerHub push

---

## 🔧 Laravel Project Prerequisites

### Laravel App Should Include:

* `.env.example` (copied to `.env` in CI)
* `artisan` CLI
* `composer.json`
* PHPUnit config (`phpunit.xml`)
* Dockerfile (for production)

### Jenkins Agent Should Have:

* PHP 8.x, Composer
* Node.js (for Dusk/UI tests)
* Docker
* MySQL/PostgreSQL if required for integration tests

### Jenkins Credentials:

* `dockerhub-username` (Secret Text)
* `dockerhub-password` (Secret Text)

---

## 📄 1. Laravel Jenkins **Pipeline Job**

### ✅ `Jenkinsfile`

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

        stage('Set Up Laravel') {
            steps {
                sh '''
                    cp .env.example .env
                    composer install --no-interaction
                    php artisan key:generate
                    php artisan config:cache
                '''
            }
        }

        stage('Linting (PHPStan, PHPCS)') {
            steps {
                sh '''
                    ./vendor/bin/phpstan analyse || true
                    ./vendor/bin/phpcs --standard=PSR12 app/ || true
                '''
            }
        }

        stage('Security Scan') {
            steps {
                sh '''
                    composer require --dev roave/security-advisories:dev-latest
                    composer update
                '''
            }
        }

        stage('Unit + Integration Tests with Coverage') {
            steps {
                sh '''
                    ./vendor/bin/phpunit --coverage-clover=coverage.xml
                '''
            }
        }

        stage('Smoke Tests') {
            steps {
                sh '''
                    php artisan serve > /dev/null 2>&1 &
                    sleep 5
                    curl -f http://127.0.0.1:8000 || exit 1
                    kill $(lsof -t -i:8000)
                '''
            }
        }

        stage('E2E Tests (Laravel Dusk or Codeception)') {
            steps {
                sh '''
                    # Example for Laravel Dusk
                    php artisan dusk || true
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t laravel-app:latest .'
            }
        }

        stage('Docker Scout Scan') {
            steps {
                sh '''
                    curl -fsSL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh -o install-scout.sh
                    sh install-scout.sh

                    docker scout quickview laravel-app:latest || true
                    docker scout cves laravel-app:latest --only-severities critical,high --output sarif > scout-report.sarif || true
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                    echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin
                    docker tag laravel-app:latest $DOCKERHUB_USER/laravel-app:latest
                    docker push $DOCKERHUB_USER/laravel-app:latest
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

## 🛠 2. Laravel Jenkins **Freestyle Job**

### Job Setup

* Git → Laravel repo
* Build Env → Secret text credentials
* Add **Build Step → Execute Shell**:

```bash
#!/bin/bash

echo "Setting up Laravel project..."
cp .env.example .env
composer install --no-interaction
php artisan key:generate
php artisan config:cache

echo "Running Linting..."
./vendor/bin/phpstan analyse || true
./vendor/bin/phpcs --standard=PSR12 app/ || true

echo "Running Security Scan..."
composer require --dev roave/security-advisories:dev-latest
composer update

echo "Running Unit + Integration Tests with Coverage..."
./vendor/bin/phpunit --coverage-clover=coverage.xml

echo "Running Smoke Test..."
php artisan serve > /dev/null 2>&1 &
sleep 5
curl -f http://127.0.0.1:8000 || exit 1
kill $(lsof -t -i:8000)

echo "Running E2E Tests..."
php artisan dusk || true
# or ./vendor/bin/codecept run acceptance

echo "Building Docker image..."
docker build -t laravel-app:latest .

echo "Installing Docker Scout..."
curl -fsSL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh -o install-scout.sh
sh install-scout.sh

echo "Running Docker Scout Scan..."
docker scout quickview laravel-app:latest || true
docker scout cves laravel-app:latest --only-severities critical,high --output sarif > scout-report.sarif || true

echo "Pushing Docker Image..."
echo "$DOCKERHUB_PASSWORD" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin
docker tag laravel-app:latest $DOCKERHUB_USERNAME/laravel-app:latest
docker push $DOCKERHUB_USERNAME/laravel-app:latest
```

* Post-build actions → Archive artifacts:

  ```
  coverage.xml, scout-report.sarif
  ```

---

## 🧱 Laravel Folder Structure Expectations

```
laravel-app/
├── app/
├── tests/
│   ├── Feature/
│   └── Unit/
├── public/
├── .env.example
├── Dockerfile
├── composer.json
├── phpunit.xml
└── artisan
```

---

## ✅ Summary Comparison

| Feature                  | Pipeline Job | Freestyle Job |
| ------------------------ | ------------ | ------------- |
| Git Checkout             | ✅            | ✅             |
| Composer Setup           | ✅            | ✅             |
| PHPStan & PHPCS          | ✅            | ✅             |
| Security Scan            | ✅            | ✅             |
| Unit + Integration Tests | ✅            | ✅             |
| Smoke Test               | ✅            | ✅             |
| Laravel Dusk UI Tests    | ✅            | ✅             |
| Docker Build & Push      | ✅            | ✅             |
| Docker Scout CVE Scan    | ✅            | ✅             |
| Archive Reports          | ✅            | ✅             |

---
