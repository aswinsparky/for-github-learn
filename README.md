# Security Scan GitHub Actions Pipeline

This document explains the "Security Scan" GitHub Actions workflow end-to-end, from prerequisites to how each step in the pipeline works. After reading this, you should be able to create or modify similar pipelines on your own.

---

## Prerequisites

Before using or modifying this pipeline, ensure the following:

### Git and GitHub Knowledge
- **Official Git Tutorial**: Learn Git fundamentals from the official Git documentation
  - Link: https://git-scm.com/docs/gittutorial
  
- **Pro Git Book** (Free, comprehensive guide)
  - Link: https://git-scm.com/book/en/v2
  
- **GitHub Official Documentation**: Get started with GitHub
  - Link: https://docs.github.com/en/get-started
  
- **GitHub Learning Resources**: Comprehensive learning paths
  - Link: https://docs.github.com/en/get-started/start-your-journey/git-and-github-learning-resources

### Repository Requirements
Your GitHub repository must contain:

1. **Application source code** (e.g., Python, Java, Node.js, etc.)
2. **Dockerfile** at the repository root
   - Used by Trivy and Hadolint for container security scanning
3. **Optional: Terraform directory**
   - If present, Terraform + Checkov scans will run automatically
   - Structure: `Terraform/environments/dev.tfvars` and `Terraform/environments/prod.tfvars`
4. **sonar-project.properties** file in the repository root
   - Contains configuration for SonarQube
   - Required fields:
     ```
     sonar.host.url=https://your-sonarqube-instance.com
     sonar.token=your-sonar-token
     sonar.projectKey=your-project-key
     ```

### GitHub Actions Setup
- GitHub Actions must be enabled for your repository
- Create the workflow file at: `.github/workflows/security-scan.yml`
- Copy the provided YAML content into this file

### AWS Configuration (if using Terraform)
- An AWS IAM role configured for GitHub Actions OIDC
  - Example role ARN: `arn:aws:iam::<account-id>:role/Githubactions`
- Necessary IAM permissions for Terraform (e.g., EC2, S3, VPC access)
- GitHub-AWS OIDC trust configured

### SonarQube Setup
- Access to SonarQube Server or SonarQube Cloud instance
  - SonarQube Server Docs: https://docs.sonarsource.com/sonarqube-server/analyzing-source-code/scanners/sonarscanner
  - SonarQube Cloud Docs: https://docs.sonarsource.com/sonarqube-cloud/advanced-setup/ci-based-analysis/sonarscanner-cli
- A project configured in SonarQube
- SonarScanner CLI version 5.0.1 will be downloaded automatically

---

## Workflow Overview

The workflow is named **"Security Scan"** and runs automatically on pull requests. Its main responsibilities:

1. **Run Static Code Analysis**
   - SonarQube (via SonarScanner CLI) - code quality and security
   - Bandit - Python security vulnerabilities

2. **Run Container and Dockerfile Scans**
   - Trivy - Dockerfile misconfigurations
   - Hadolint - Dockerfile best practices

3. **Run Infrastructure-as-Code Scans** (Optional)
   - Terraform plan generation
   - Checkov - Terraform security and compliance

4. **Aggregate Results**
   - Human-readable Markdown security report
   - JSON findings file for inline PR comments

5. **Post Results to Pull Request**
   - Inline comments on specific lines where issues are found
   - Single, updateable summary comment with full report

The workflow is defined in: `.github/workflows/security-scan.yml`

---

## Trigger and Permissions

### Workflow Name and Trigger

```yaml
name: Security Scan

on:
  pull_request:
    types: [opened, synchronize, reopened]
```

**Explanation:**
- The pipeline runs whenever a pull request is:
  - **opened**: First created
  - **synchronize**: Updated with new commits
  - **reopened**: Reopened after being closed
- This ensures new code is always scanned before merging

### Permissions

```yaml
permissions:
  contents: read
  pull-requests: write
  id-token: write
```

**Explanation:**

| Permission | Purpose |
|-----------|---------|
| `contents: read` | Allows reading repository code for scanning |
| `pull-requests: write` | Allows posting and updating comments on PRs |
| `id-token: write` | Required for GitHub OIDC to assume AWS IAM roles securely |

---

## Job Configuration

### Job Setup

```yaml
jobs:
  security-scan:
    if: |
      (startsWith(github.head_ref, 'feature/') && github.base_ref == 'develop') || 
      (github.head_ref == 'develop' && github.base_ref == 'main')
    runs-on: ubuntu-latest
```

**Explanation:**

- **Job Name**: `security-scan`
- **Conditional Execution**: Pipeline only runs when:
  - A `feature/*` branch targets `develop`, OR
  - `develop` targets `main`
- **Runner**: `ubuntu-latest` (Ubuntu virtual machine provided by GitHub)

**Benefits of conditional execution:**
- Reduces unnecessary scanning on non-critical PRs
- Focuses quality gates on key integration points
- Saves GitHub Actions minutes/costs

---

## Step-by-Step Explanation

### Step 1: Checkout Code

```yaml
- name: Checkout Code
  uses: actions/checkout@v3
```

**What it does:**
- Clones the repository code into the GitHub Actions runner
- Makes all files available for scanning in subsequent steps
- Uses version 3 of the official checkout action

---

### Step 2: Configure AWS Credentials

```yaml
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::302263040839:role/Githubactions
    aws-region: ap-south-1
```

**What it does:**
- Uses GitHub's OIDC (OpenID Connect) token to assume an AWS IAM role
- Sets up temporary AWS credentials (no long-lived keys)
- Makes credentials available for tools like Terraform and Checkov
- Configures AWS region to `ap-south-1` (Asia Pacific - Mumbai)

**Security benefit:** Uses short-lived credentials instead of storing AWS keys in GitHub

---

### Step 3: Install General Prerequisites

```yaml
- name: Install prerequisites
  run: sudo apt-get update && sudo apt-get install -y jq curl wget
```

**What it does:**
Installs three essential tools:

| Tool | Usage in Pipeline |
|------|-------------------|
| `jq` | Command-line JSON processor - parses scan results JSON files |
| `curl` | HTTP client - calls SonarQube API to fetch issues |
| `wget` | File downloader - downloads SonarScanner CLI and Hadolint binaries |

---

## Section A: SonarQube Static Code Analysis

SonarQube provides comprehensive code quality and security analysis.

### A.1: Install SonarScanner CLI

```yaml
- name: Install SonarScanner CLI
  run: |
    wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
    unzip sonar-scanner-cli-5.0.1.3006-linux.zip
    mv sonar-scanner-5.0.1.3006-linux sonar-scanner
    echo "$PWD/sonar-scanner/bin" >> $GITHUB_PATH
```

**What it does:**
1. Downloads SonarScanner CLI version 5.0.1 (Linux binary)
2. Unzips the downloaded archive
3. Renames directory to `sonar-scanner`
4. Adds `sonar-scanner/bin` to PATH so `sonar-scanner` command is available globally

**Reference:** https://docs.sonarsource.com/sonarqube-server/analyzing-source-code/scanners/sonarscanner

---

### A.2: Run SonarQube Scan

```yaml
- name: Run SonarQube Scan (using sonar-project.properties)
  run: |
    sonar-scanner
```

**What it does:**
- Executes `sonar-scanner` command
- Automatically reads configuration from `sonar-project.properties`:
  - `sonar.host.url` - SonarQube server URL
  - `sonar.token` - Authentication token
  - `sonar.projectKey` - Project identifier
- Analyzes code and uploads results to SonarQube

---

### A.3: Fetch SonarQube Issues

```yaml
- name: Fetch SonarQube Issues (read settings from sonar-project.properties)
  run: |
    SONAR_HOST_URL=$(grep '^sonar.host.url=' sonar-project.properties | cut -d'=' -f2-)
    SONAR_TOKEN=$(grep '^sonar.token=' sonar-project.properties | cut -d'=' -f2-)
    SONAR_PROJECT_KEY=$(grep '^sonar.projectKey=' sonar-project.properties | cut -d'=' -f2-)

    echo "HOST: $SONAR_HOST_URL"
    echo "PROJECT: $SONAR_PROJECT_KEY"

    curl -s -u "${SONAR_TOKEN}:" \
      "$SONAR_HOST_URL/api/issues/search?componentKeys=$SONAR_PROJECT_KEY&ps=500" > sonar_issues.json
    echo "Sonar issues JSON (total, count):"
    jq '.total, .issues | length' sonar_issues.json || true
```

**What it does:**
1. **Reads configuration** from `sonar-project.properties` using `grep` and `cut`:
   - Extracts `SONAR_HOST_URL`, `SONAR_TOKEN`, `SONAR_PROJECT_KEY`
2. **Calls SonarQube API** using `curl`:
   - Endpoint: `/api/issues/search`
   - Parameters: `componentKeys`, `ps=500` (page size 500)
   - Authentication: Bearer token
3. **Saves results** to `sonar_issues.json`
4. **Prints summary** using `jq`:
   - Total issue count
   - Number of individual issues

**Output file:** `sonar_issues.json` (used later for report and inline comments)

---

## Section B: Bandit - Python Security Scanner

Bandit identifies common security issues in Python code.

```yaml
- name: Run Bandit Scan
  run: |
    pip install bandit
    bandit -r . -f json -o bandit-report.json || true
```

**What it does:**
1. **Installs Bandit** via pip
2. **Runs security scan**:
   - `-r .` - Recursively scan current directory
   - `-f json` - Output format as JSON
   - `-o bandit-report.json` - Save to file
3. **|| true** - Ensures step succeeds even if vulnerabilities found (allows pipeline to continue)

**Security issues Bandit detects:**
- Hard-coded passwords and secrets
- Insecure SQL queries
- Dangerous cryptographic functions
- Insecure temporary files
- And more...

**Output file:** `bandit-report.json`

---

## Section C: Trivy - Container Misconfiguration Scanner

Trivy scans Dockerfiles for security misconfigurations.

### C.1: Install Trivy

```yaml
- name: Install Trivy
  run: |
    sudo apt-get update
    sudo apt-get install -y wget apt-transport-https gnupg lsb-release
    wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
    echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -cs) main" | \
      sudo tee /etc/apt/sources.list.d/trivy.list
    sudo apt-get update
    sudo apt-get install -y trivy
```

**What it does:**
- Adds Aqua Security's official Trivy repository to apt sources
- Downloads and adds GPG key for repository verification
- Installs Trivy package

---

### C.2: Run Trivy Dockerfile Scan

```yaml
- name: Run Trivy Dockerfile Scan
  run: |
    trivy config --severity HIGH,CRITICAL --format json --output trivy-report.json Dockerfile || true
```

**What it does:**
1. **Scans Dockerfile** for misconfigurations
2. **Filters for severity**: Only HIGH and CRITICAL issues
3. **Output format**: JSON
4. **Output file**: `trivy-report.json`
5. **|| true**: Continues even if issues found

**Common issues Trivy finds:**
- Running as root user
- Missing HEALTHCHECK
- Using latest tag instead of pinned versions
- Running privileged containers unnecessarily

**Output file:** `trivy-report.json`

---

## Section D: Hadolint - Dockerfile Linter

Hadolint enforces Dockerfile best practices and standards.

### D.1: Install Hadolint

```yaml
- name: Install Hadolint
  run: |
    wget -O hadolint https://github.com/hadolint/hadolint/releases/latest/download/hadolint-Linux-x86_64
    chmod +x hadolint
    sudo mv hadolint /usr/local/bin/hadolint
```

**What it does:**
1. **Downloads** latest Hadolint binary for Linux x86_64
2. **Makes executable** with `chmod +x`
3. **Moves to PATH** so it's globally accessible

---

### D.2: Run Hadolint Scan

```yaml
- name: Run Hadolint Scan
  run: |
    hadolint --format json Dockerfile > hadolint-report.json || true
```

**What it does:**
1. **Scans Dockerfile** for best-practice violations
2. **Output format**: JSON
3. **Output file**: `hadolint-report.json`
4. **|| true**: Continues pipeline even with issues

**Common issues Hadolint finds:**
- Missing metadata labels
- Using `RUN` with multiple commands without `&&`
- Inefficient layer caching
- Shell form instead of exec form for ENTRYPOINT

**Output file:** `hadolint-report.json`

---

## Section E: Terraform and Checkov (Conditional)

These steps only run if a `Terraform` directory exists in the repository.

### E.1: Check if Terraform Folder Exists

```yaml
- name: Check if Terraform folder exists
  id: check_terraform
  run: |
    if [ -d "Terraform" ]; then
      echo "terraform_exists=true" >> $GITHUB_OUTPUT
      echo "✅ Terraform folder found"
    else
      echo "terraform_exists=false" >> $GITHUB_OUTPUT
      echo "⚠️  Terraform folder not found, skipping Terraform/Checkov scans"
    fi
```

**What it does:**
- Checks if `Terraform` directory exists
- Sets output variable `terraform_exists` (true/false)
- All following Terraform steps use this condition to skip if not found

**Key concept:** Using `id: check_terraform` allows other steps to reference this output with `steps.check_terraform.outputs.terraform_exists`

---

### E.2: Set Environment for Terraform Plan

```yaml
- name: Set environment for Terraform plan
  id: set_env
  if: steps.check_terraform.outputs.terraform_exists == 'true'
  run: |
    if [[ "${{ github.head_ref }}" == feature/* && "${{ github.base_ref }}" == "develop" ]]; then
      echo "TFVARS=dev.tfvars" >> $GITHUB_ENV
    elif [[ "${{ github.head_ref }}" == "develop" && "${{ github.base_ref }}" == "main" ]]; then
      echo "TFVARS=prod.tfvars" >> $GITHUB_ENV
    else
      echo "TFVARS=dev.tfvars" >> $GITHUB_ENV
    fi
```

**What it does:**
- **Determines environment** based on PR branches:
  - `feature/*` → `develop`: Use `dev.tfvars`
  - `develop` → `main`: Use `prod.tfvars`
  - Default: Use `dev.tfvars`
- **Sets environment variable** `TFVARS` for use in later steps
- **Conditional execution** only if Terraform folder exists

**Directory structure expected:**
```
Terraform/
├── environments/
│   ├── dev.tfvars
│   └── prod.tfvars
├── main.tf
├── variables.tf
└── ...
```

---

### E.3: Set up Terraform

```yaml
- name: Set up Terraform
  if: steps.check_terraform.outputs.terraform_exists == 'true'
  uses: hashicorp/setup-terraform@v3
```

**What it does:**
- Uses HashiCorp's official Terraform setup action
- Installs appropriate version of Terraform CLI
- Makes `terraform` command available

---

### E.4: Terraform Init

```yaml
- name: Terraform Init
  if: steps.check_terraform.outputs.terraform_exists == 'true'
  run: terraform init
  working-directory: Terraform
```

**What it does:**
- Initializes Terraform configuration
- Downloads required providers
- Sets up backend
- **working-directory**: Runs command in `Terraform/` folder

---

### E.5: Terraform Plan

```yaml
- name: Terraform Plan
  if: steps.check_terraform.outputs.terraform_exists == 'true'
  run: terraform plan -var-file=./environments/${TFVARS} -out=tfplan.binary
  working-directory: Terraform
```

**What it does:**
1. **Generates execution plan** for proposed infrastructure changes
2. **Uses environment-specific variables** from `./environments/${TFVARS}`:
   - `dev.tfvars` or `prod.tfvars` depending on branches
3. **Outputs binary plan** to `tfplan.binary`
4. **working-directory**: Runs in `Terraform/` folder

**Why binary format?** Preserves all plan metadata for accurate Checkov analysis

---

### E.6: Convert Plan to JSON

```yaml
- name: Convert Plan to JSON
  if: steps.check_terraform.outputs.terraform_exists == 'true'
  run: terraform show -json tfplan.binary > tfplan.json
  working-directory: Terraform
```

**What it does:**
- **Converts** binary Terraform plan to human-readable and machine-parseable JSON
- **Output file**: `tfplan.json` (used by Checkov)
- **working-directory**: Runs in `Terraform/` folder

---

### E.7: Install Checkov

```yaml
- name: Install Checkov
  if: steps.check_terraform.outputs.terraform_exists == 'true'
  run: pip install checkov
```

**What it does:**
- Installs Checkov, a static analysis tool for Infrastructure-as-Code
- Installs latest version via pip

---

### E.8: Run Checkov

```yaml
- name: Run Checkov (scan plan with enrichment)
  if: steps.check_terraform.outputs.terraform_exists == 'true'
  run: checkov -f tfplan.json --repo-root-for-plan-enrichment . -o json > checkov-report.json || true
  working-directory: Terraform
```

**What it does:**
1. **Scans Terraform plan** (`tfplan.json`)
2. **Enriches with repo context** (`--repo-root-for-plan-enrichment .`):
   - Provides better understanding of checks
   - Links violations to actual resource definitions
3. **Output format**: JSON
4. **Output file**: `checkov-report.json`
5. **|| true**: Continues even with violations found
6. **working-directory**: Runs in `Terraform/` folder

**Security checks Checkov performs:**
- Unencrypted storage
- Public access rules
- Missing security groups
- Logging not enabled
- And 1000+ more checks...

**Output file:** `Terraform/checkov-report.json`

---

## Section F: Format Aggregated Report

This complex step aggregates all scan results into a human-readable Markdown report.

```yaml
- name: Format Reports
  run: |
    # Creates the main report file
    echo "# :shield: Security Scan Report" > report.md
    echo "" >> report.md

    # Reads all scan JSON files
    # Computes summary statistics for each tool
    # ... (complex bash + jq processing)
    # Generates detailed sections for each tool with findings
```

**What this step does (overview):**

1. **Creates header** with shield emoji
2. **Extracts metrics** from each scan file:
   - Checkov: passed, failed, skipped counts
   - SonarQube: total issues
   - Bandit: total findings
   - Trivy: HIGH/CRITICAL misconfigurations
   - Hadolint: total issues

3. **Generates summary section**:
   - Quick overview of all tools
   - Status emojis (🟢 passing, 🔴 failing)
   - Overall assessment

4. **Generates detailed sections**:
   - **Checkov Section**: Check ID, name, severity, guideline, description, GitHub links
   - **SonarQube Section**: Severity, type, file, line, message, GitHub links
   - **Bandit Section**: Issue text, file, line, severity, confidence, GitHub links
   - **Hadolint Section**: Line number, code, severity, message, GitHub link
   - **Trivy Section**: ID, type, severity, message, resolution, GitHub link

5. **Links to GitHub**:
   - Every finding includes a clickable link to the exact line in GitHub
   - Uses repo URL and commit SHA to create direct links
   - Example: `https://github.com/owner/repo/blob/branch/file.py#L42`

6. **Footer**: 
   - Note that report was auto-generated
   - Timestamp information

**Output file:** `report.md` (posted as PR comment summary)

**Key insight:** This report is designed to be both:
- **Human-readable**: Easy to understand for developers
- **Machine-parseable**: Can be used by other tools
- **Interactive**: Direct links to problematic code

---

## Section G: Prepare Findings for Inline Comments

This step transforms all scan results into a unified format for posting inline PR comments.

```yaml
- name: Prepare findings JSON for batched PR comments
  run: |
    # Creates intermediate JSON files for each tool
    # Transforms each tool's format into unified schema:
    # {
    #   "path": "relative/file/path.py",
    #   "line": 42,
    #   "body": "**[SonarQube][CRITICAL]** Message about issue"
    # }
```

**What this step does:**

1. **Initializes empty arrays** for each tool's findings:
   - `sonar_findings.json`
   - `bandit_findings.json`
   - `trivy_findings.json`
   - `hadolint_findings.json`
   - `checkov_findings.json`

2. **Processes each tool's JSON output**:

   **SonarQube:**
   - Reads `sonar_issues.json`
   - Maps issue to source file path and line
   - Creates message: `🔍 **[SonarQube][CRITICAL][BUG]** Issue message`

   **Bandit:**
   - Reads `bandit-report.json`
   - Strips leading `./` from filename
   - Creates message: `🐍 **[Bandit][HIGH][CONFIDENCE]** Issue text`

   **Trivy:**
   - Reads `trivy-report.json`
   - Uses Dockerfile as path, cause metadata for line
   - Creates message: `🐳 **[Trivy][CRITICAL][DL3009]** Message | Resolution: ...`

   **Hadolint:**
   - Reads `hadolint-report.json`
   - Uses Dockerfile and line number
   - Creates message: `🐋 **[Hadolint][ERROR][DL3006]** Message`

   **Checkov:**
   - Reads `Terraform/checkov-report.json`
   - Constructs path as `Terraform/<file_path>`
   - Creates message: `☁️ **CKV_AWS_001: Ensure S3 bucket has public access blocked**`

3. **Merges all findings** into single array:
   - Uses `jq -s 'add'` to concatenate arrays
   - Output: `all_findings.json`

4. **Validates output**:
   - Checks that `all_findings.json` is valid JSON array
   - Prints total count and sample finding

5. **Cleans up** intermediate files

**Output file:** `all_findings.json`

**Format of each finding:**
```json
{
  "path": "app/main.py",
  "line": 42,
  "body": "🔍 **[SonarQube][CRITICAL][SQL_INJECTION]** Potential SQL injection vulnerability"
}
```

---

## Section H: Post Inline Review Comments (Batched with Retry)

This step posts inline comments on the PR for each finding, but only on lines that are actually changed.

### Key Features:

1. **Filters to PR diff**: Only comments on lines changed in the PR
2. **Batches requests**: Posts up to 20 comments per API call (GitHub rate limits)
3. **Implements retry logic**: Exponential backoff for rate limits and server errors
4. **Smart validation**: Parses patch headers to identify valid comment lines

```yaml
- name: Post inline review comments with batching and retry
  if: github.event_name == 'pull_request'
  uses: actions/github-script@v7
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    script: |
      # JavaScript code that:
      # 1. Reads all_findings.json
      # 2. Fetches PR file list and diffs
      # 3. Builds map of valid lines per file
      # 4. Filters findings to only those in PR diff
      # 5. Posts comments in chunks with retry
```

**What this step does:**

1. **Reads findings**: Loads `all_findings.json`
2. **Validates data**: Ensures it's a valid JSON array
3. **Fetches PR metadata**:
   - Commit SHA from `context.payload.pull_request.head.sha`
   - PR number and repo info
4. **Fetches PR files**: Calls GitHub API to get all files in PR
5. **Parses diffs**: For each file, determines which lines are "in the diff":
   - Lines with `+` (added) or ` ` (context) in unified diff format
   - Excludes `-` (removed) lines
6. **Filters findings**:
   - Only comment if file exists in PR
   - Only comment if line is in the diff
   - Skips findings outside PR context
7. **Builds comments array**:
   ```json
   {
     "path": "app/main.py",
     "line": 42,
     "side": "RIGHT",
     "body": "Issue description"
   }
   ```
8. **Posts in chunks**:
   - Groups findings into chunks of 20
   - Uses `pulls.createReview()` API call per chunk
   - Adds 3-second delay between chunks
9. **Implements retry logic**:
   - Catches secondary rate limit errors (HTTP 429)
   - Implements exponential backoff: 3s → 6s → 12s → 24s → 48s
   - Maximum 5 retry attempts
   - Skips comments that fail due to "line not in diff" errors
10. **Reports summary**: Prints success and failure counts

**Why batching matters:**
- GitHub API rate limits concurrent comments
- Batching 20 comments per call is more efficient than 1 per call
- Delays between batches prevent hitting secondary rate limits

**Why filtering to PR diff matters:**
- Commenting on unchanged lines would be off-topic
- Only changes made in the PR are relevant to reviewers
- Respects GitHub's "line must be part of diff" validation

---

## Section I: Post (or Update) Security Scan Summary

This step ensures there's always a single, up-to-date summary comment on the PR.

```yaml
- name: Post Security Scan Summary to PR (with update/reuse)
  if: github.event_name == 'pull_request'
  uses: actions/github-script@v7
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    script: |
      # JavaScript code that:
      # 1. Reads report.md
      # 2. Adds marker comment for identification
      # 3. Checks if summary comment already exists
      # 4. Updates existing or creates new
```

**What this step does:**

1. **Reads report file**: Loads `report.md`
2. **Adds marker**: Wraps report with HTML comment marker
   ```markdown
   <!-- security-scan-full-report -->
   # Shield: Security Scan Report
   ...
   ```
3. **Lists existing comments**: Calls GitHub API to get all PR comments
4. **Searches for existing marker**: Finds comment with `<!-- security-scan-full-report -->`
5. **Decision logic**:
   - **If marker found**: Update that comment with latest report
   - **If not found**: Create new comment with report
6. **Implements retry**: Catches and retries on rate limit errors
7. **Result**: PR always has exactly ONE summary comment with latest results

**Benefits:**
- No duplicate summary comments cluttering PR
- Latest scan results always visible in same place
- Easy for developers to find comprehensive report
- Previous reports automatically replaced

---

## How to Create a Similar Pipeline

### Step 1: Plan Your Security Scanning Strategy

Decide what you want to scan and when:

**Application Code Scanning:**
- SonarQube for code quality and security
- Bandit for Python vulnerabilities
- Add ESLint, Checkmarx, or SpotBugs for other languages

**Container Scanning:**
- Trivy for Dockerfile misconfigurations
- Hadolint for Dockerfile best practices
- Optional: Scan for vulnerable dependencies

**Infrastructure Scanning:**
- Terraform + Checkov for IaC security
- CloudFormation scanning if using AWS CloudFormation
- Ansible scanning if using Ansible

**Timing:**
- Every PR (most common - catch issues early)
- Before merging to main
- Schedule: daily/weekly on main branch

### Step 2: Create Workflow File

Create file: `.github/workflows/security-scan.yml`

```yaml
name: Security Scan

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  security-scan:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      # Add steps from sections below
```

### Step 3: Add Core Steps (Minimum)

Start with these essentials:

```yaml
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Install prerequisites
        run: sudo apt-get update && sudo apt-get install -y jq curl wget

      # Add your specific scanning tools here
```

### Step 4: Add Security Tools

**For Python projects, add Bandit:**
```yaml
      - name: Run Bandit Scan
        run: |
          pip install bandit
          bandit -r . -f json -o bandit-report.json || true
```

**For code quality, add SonarQube:**
```yaml
      - name: Install SonarScanner CLI
        run: |
          wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
          unzip sonar-scanner-cli-5.0.1.3006-linux.zip
          mv sonar-scanner-5.0.1.3006-linux sonar-scanner
          echo "$PWD/sonar-scanner/bin" >> $GITHUB_PATH

      - name: Run SonarQube Scan
        run: sonar-scanner
```

**For Docker, add Trivy:**
```yaml
      - name: Install and Run Trivy
        run: |
          sudo apt-get install -y trivy
          trivy config --severity HIGH,CRITICAL --format json --output trivy-report.json Dockerfile || true
```

### Step 5: Configure External Services

**For SonarQube:**
1. Create `sonar-project.properties` in repo root:
   ```properties
   sonar.host.url=https://your-sonarqube.com
   sonar.token=your-token-here
   sonar.projectKey=project-key
   sonar.sources=src
   ```
2. Reference: https://docs.sonarsource.com/sonarqube-server/analyzing-source-code/scanners/sonarscanner

**For AWS (if using Terraform):**
1. Set up GitHub OIDC identity provider in AWS
2. Create IAM role with proper trust policy
3. Add role ARN to workflow:
   ```yaml
   role-to-assume: arn:aws:iam::ACCOUNT-ID:role/Githubactions
   ```

### Step 6: Add Reporting (Optional but Recommended)

**Generate Markdown report:**
```yaml
      - name: Format Reports
        run: |
          echo "# Security Scan Report" > report.md
          # Add tool results to report
```

**Post results:**
```yaml
      - name: Post Security Scan Summary to PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('report.md', 'utf8');
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: report
            });
```

### Step 7: Test and Iterate

1. **Push workflow file** to repository
2. **Create test PR** to verify workflow runs
3. **Check workflow logs** for errors
4. **Fix issues incrementally**:
   - Start with one tool
   - Verify it works
   - Add next tool
   - Repeat
5. **Gradually enhance**:
   - Add more scanning tools
   - Improve reporting
   - Add inline comments
   - Fine-tune filtering

### Step 8: Customize for Your Needs

Common customizations:

**Only scan specific branches:**
```yaml
on:
  pull_request:
    branches: [main, develop]
```

**Only scan when specific files change:**
```yaml
on:
  pull_request:
    paths:
      - 'src/**'
      - 'Terraform/**'
      - 'Dockerfile'
```

**Fail the build if critical issues found:**
```yaml
      - name: Check for critical issues
        run: |
          CRITICAL=$(jq '.results | map(select(.severity=="CRITICAL")) | length' bandit-report.json)
          if [ "$CRITICAL" -gt 0 ]; then
            echo "Critical issues found!"
            exit 1
          fi
```

**Send notifications:**
```yaml
      - name: Notify on failure
        if: failure()
        run: |
          # Send Slack message, email, etc.
```

---

## Summary

The Security Scan pipeline demonstrates:

✅ **Comprehensive Security Coverage**
- Application code analysis (SonarQube, Bandit)
- Container scanning (Trivy, Hadolint)
- Infrastructure scanning (Checkov)

✅ **Developer-Friendly Feedback**
- Inline comments on exact problem lines
- Summary report with links to code
- Clear, actionable messages

✅ **Production-Ready Features**
- Error handling and retries
- Rate limit management
- Conditional execution
- Report updates (not duplicates)

✅ **Extensible Architecture**
- Easy to add new scanning tools
- Modular step structure
- Reusable patterns

By understanding each section of this pipeline, you can:
- Build custom security workflows for your projects
- Integrate new scanning tools
- Adapt to your organization's specific needs
- Scale security scanning across your team

---

## Additional Resources

**Git & GitHub Learning:**
- Official Git Tutorial: https://git-scm.com/docs/gittutorial
- Pro Git Book: https://git-scm.com/book/en/v2
- GitHub Documentation: https://docs.github.com/en/get-started

**Security Scanning Tools:**
- SonarQube Server: https://docs.sonarsource.com/sonarqube-server/analyzing-source-code/scanners/sonarscanner
- SonarQube Cloud: https://docs.sonarsource.com/sonarqube-cloud/advanced-setup/ci-based-analysis/sonarscanner-cli
- Bandit GitHub: https://github.com/PyCQA/bandit
- Trivy GitHub: https://github.com/aquasecurity/trivy
- Hadolint GitHub: https://github.com/hadolint/hadolint
- Checkov GitHub: https://github.com/bridgecrewio/checkov

**GitHub Actions:**
- GitHub Actions Documentation: https://docs.github.com/en/actions
- Workflow Syntax: https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions
- GitHub Script Action: https://github.com/actions/github-script

---

**Document Version:** 1.0
**Last Updated:** December 2025
**Audience:** Developers, DevOps Engineers, Security Engineers
