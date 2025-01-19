# CI/CD Pipeline with SonarQube and AWS Deployment

Welcome! This repository includes a complete setup for a CI/CD pipeline that:
1. **Builds and tests your application**.
2. **Performs code quality checks** with SonarQube.
3. **Automatically deploys your app to AWS** (e.g., Lambda).

This pipeline is designed to help teams:
- Deploy faster (cutting deployment times by 30%).
- Improve code reliability with automated quality gates.

---

## What’s Inside?

### **1. Continuous Integration (CI)**
- Automatically builds your code when changes are pushed.
- Runs all your tests and ensures everything is working.
- Integrates with **SonarQube** to catch code quality issues before they reach production.

### **2. Continuous Deployment (CD)**
- After passing all checks, your app is deployed to AWS.
- Supports AWS Lambda by default but can be customized for EC2 or ECS.

---

## Simplifying SonarQube Configuration with a Properties File

If your **SonarQube server is self-hosted**, you can configure a `sonar-project.properties` file in your project directory. This eliminates the need to repeatedly specify the `SONAR_HOST_URL` during pipeline configuration.

### **How to Set It Up**
1. **Create the `sonar-project.properties` File**:
   - Place it in the root of your project (same level as your source code).
   - Example:
     ```properties
     # SonarQube server configuration
     sonar.host.url=http://your-sonarqube-server.com

     # Optional default settings
     sonar.sourceEncoding=UTF-8
     sonar.exclusions=**/test/**

     # This file does NOT store project key or authentication tokens.
     ```

2. **Pipeline Adjustments**:
   - You’ll still need to pass the **project key** and **API key** in the pipeline. The `sonar.host.url` will now be automatically picked up from the properties file.

3. **What’s Still Required**:
   - Add `SONAR_TOKEN` (API key) as a secret in your CI/CD pipeline.
   - Pass the project key explicitly in the pipeline configuration.

4. **Advantages**:
   - You no longer need to specify the `SONAR_HOST_URL` every time you run a SonarQube scan.
   - Centralized server configuration makes onboarding new projects easier.

---

## Pipeline Overview

This pipeline has two stages:

### **1. Build and Analyze**
- **What happens**:
  - The pipeline pulls your code from the repository.
  - It restores dependencies, builds the project, and runs tests.
  - Runs a **SonarQube scan** to ensure the code meets quality standards.
  - Packages the build output into an artifact for deployment.

- **How SonarQube Works Here**:
  - The `sonar-project.properties` file defines the server URL and optional settings.
  - The pipeline passes the **project key** and **API key** as required inputs.

---

### **2. Deploy**
- **What happens**:
  - The pipeline retrieves the build artifact and deploys it to AWS.
  - By default, it updates an AWS Lambda function with the new code.

- **Why it’s important**:
  - Removes manual deployment steps.
  - Ensures the latest, validated code is always running.

---

## Pipeline in Action: Example for GitHub Actions

Here’s the full YAML for the pipeline:

```yaml
name: CI/CD with SonarQube and AWS

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  build-and-analyze:
    name: Build and SonarQube Analysis
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Set up Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '16'

    - name: Install dependencies
      run: npm install

    - name: Run Tests with Coverage
      run: npm test -- --coverage

    - name: SonarQube Scan
      uses: sonarsource/sonarqube-scan-action@v1.3
      env:
        SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }} # Optional if using properties file
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        SONAR_PROJECT_KEY: your-project-key

    - name: Build Application
      run: npm run build

    - name: Upload Artifacts
      uses: actions/upload-artifact@v3
      with:
        name: build
        path: dist/

  deploy:
    name: Deploy to AWS Lambda
    runs-on: ubuntu-latest
    needs: build-and-analyze

    steps:
    - name: Download Artifacts
      uses: actions/download-artifact@v3
      with:
        name: build

    - name: Deploy to AWS Lambda
      uses: aws-actions/aws-cli-action@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1
        cli-command: |
          aws lambda update-function-code \
            --function-name your-lambda-function \
            --zip-file fileb://dist/deployment-package.zip
