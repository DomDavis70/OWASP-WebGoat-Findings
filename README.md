# Automated OWASP WebGoat Vulnerability Scanning with ZAP DAST

Welcome to the project! This CI pipeline automatically scans OWASP WebGoat for vulnerabilities using ZAP DAST (Dynamic Application Security Testing). It’s all automated via GitHub Actions and Docker, ensuring a smooth and consistent process whenever changes are pushed to the main branch.

## Overview

The pipeline does the following:
- Builds the project using Maven.
- Scans the code for security flaws with Semgrep.
- Runs WebGoat in a Docker container.
- Performs a ZAP scan on the running app.
- Archives the results of the ZAP scan.

## How It Works

### 1. **Build & Static Analysis**
The pipeline first builds the Java project with Maven and then scans the code for security issues using Semgrep.

- **Checkout the code**.
- **Set up JDK 17** (for building the project).
- **Build the project using Maven**.
- **Upload the build artifact** (WebGoat app) so it can be downloaded in the next step to build the Docker container.
- **Run Semgrep scans** to check for security vulnerabilities both in the code and its dependencies.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven
      - name: Build with Maven
        run: mvn -B package --file pom.xml
      - uses: actions/upload-artifact@v4
        with:
          name: build-artifact
          path: target/

  semgrep:
    runs-on: ubuntu-20.04
    needs: build
    container:
      image: semgrep/semgrep
    steps:
      - uses: actions/checkout@v3
      - name: Run Semgrep scans
        run: |
          semgrep ci --code
          semgrep ci --supply-chain
```

### 2. **Run WebGoat in Docker**
Once the project is built, the next step is to run the WebGoat app in a Docker container. This ensures the application is ready for testing with ZAP.

- **Download the build artifact** (WebGoat app).
- **Build and run the Docker container** with the WebGoat app.
- **Wait for WebGoat to initialize**.

```yaml
  zap-scan:
    runs-on: ubuntu-20.04
    needs: build
    steps:
      - name: Checkout repository and download artifact
        uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          name: build-artifact
          path: target/

      - name: Build and run Docker container with WebGoat
        run: |
          docker build -t webgoat-build:latest .
          docker run -d -p 127.0.0.1:8081:8080 -p 127.0.0.1:9091:9090 webgoat-build:latest

      - name: Wait for WebGoat to initialize
        run: sleep 30
```
In this step, the pipeline first downloads the WebGoat app artifact that was built earlier. Then, it builds the Docker image and runs the container on localhost (with the necessary ports mapped). The sleep 30 command is used to give WebGoat enough time to initialize before proceeding to the ZAP scan.

### 3. **ZAP Vulnerability Scanning**
Once WebGoat is running in the Docker container, we proceed with scanning the app for vulnerabilities using OWASP ZAP.

- **Run ZAP DAST Scan**: This scan will look for runtime vulnerabilities in the application.
- **Run ZAP Automation Framework Scan**: This deeper scan checks for additional vulnerabilities.
- **Archive the ZAP scan results** for easy access.

```yaml
      - name: Run ZAP DAST scan
        uses: zaproxy/action-full-scan@v0.10.0
        with:
          token: ${{ secrets.PAT }}
          docker_name: 'ghcr.io/zaproxy/zaproxy:stable'
          target: 'http://127.0.0.1:8081/WebGoat/login'
          rules_file_name: 'zap/rules.tsv'
          cmd_options: '-a'

      - name: Run ZAP Automation Framework scan
        uses: zaproxy/action-af@v0.1.0
        with:
          plan: 'zap/automation_plan.yaml'
          dir: '/zap/wrk/output'

      - name: Archive results
        uses: actions/upload-artifact@v4
        with:
          name: zap-scan-results
          path: zap-report.html
```
# Conclusion
This CI pipeline effectively automates the process of vulnerability scanning for the WebGoat application. 
