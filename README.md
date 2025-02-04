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
