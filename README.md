# GitLab
# GitLab CI/CD:
## What is CI/CD?

**CI/CD** stands for **Continuous Integration/Continuous Deployment**. It's an automated system that continuously builds, tests, and deploys your code as you make changes. Think of it as an automatic quality checker that catches bugs early before code reaches production.

---

## The 4 Steps to Get Started

### **Step 1: Create a Pipeline Configuration File**

Create a file called `.gitlab-ci.yml` in your project's root folder. This file tells GitLab what tasks to run and when.

**Key concepts:**
- **Stages**: The order in which things happen (e.g., build → test → deploy)
- **Jobs**: The actual tasks that run in each stage (e.g., compile code, run tests)

**Example workflow:**
```
Build stage: Compile your code
  ↓
Test stage: Run automated tests
  ↓
Deploy stage: Upload to production
```

---

### **Step 2: Set Up Runners**

**Runners** are machines that execute your jobs. They're like workers doing the actual work.

**Your options:**
- Use GitLab.com's built-in runners (easiest option)
- Register your own runner if you need custom setup
- Use a runner on your local machine

---

### **Step 3: Add Variables & Expressions**

**Variables** store settings and secrets (passwords, API keys) that jobs need.

**Two types:**
- **Custom variables**: You create them
- **Predefined variables**: GitLab automatically sets them with info about your pipeline

**Security features:**
- **Protected variables**: Only run on specific branches
- **Masked variables**: Hide sensitive values in logs

**Expressions** let you dynamically control pipeline behavior based on conditions (like "only run on main branch").

---

### **Step 4: Use Reusable Components**

**CI/CD components** are pre-built, reusable pipeline chunks you can share across projects.

**Benefits:**
- Reduces copy-pasting configuration
- Keeps all projects consistent
- Easier to maintain

You can publish components to GitLab's **CI/CD Catalog** to share them with your team or the community.

---

## Quick Takeaway

1. **Write rules** (.gitlab-ci.yml) → 2. **Get workers** (runners) → 3. **Add settings** (variables) → 4. **Reuse code** (components)

That's it! GitLab handles the rest automatically.

---

# GitLab CI/CD Pipelines: Complete Guide
## Architecture, Concepts, Capabilities & Limitations

---

## 1. Pipeline Overview

### What is a Pipeline?

A **GitLab CI/CD Pipeline** is an automated workflow that executes a series of sequential stages. Each stage contains one or more jobs that can run in parallel. Pipelines are triggered by code changes, merge requests, schedules, or manual requests.

**Key Characteristics:**
- A collection of sequential stages defined in `.gitlab-ci.yml`
- Executes based on Git events (push, merge request, schedule, or manual trigger)
- Automated CI/CD workflow for build, test, and deployment
- Provides feedback on code quality and readiness

### Pipeline Lifecycle

| Status | Description |
|--------|-------------|
| **Created** | Pipeline initiated but stages haven't started |
| **Running** | At least one job is actively running |
| **Success** | All stages and jobs completed successfully |
| **Failed** | At least one job failed; subsequent stages stop |
| **Canceled** | Pipeline stopped manually before completion |
| **Skipped** | Pipeline not executed due to conditions or rules |

### Deleting Pipelines

⚠️ **Important:** Deleting a pipeline does **NOT** automatically delete child pipelines
- Child pipelines must be deleted separately if needed
- Orphaned child pipelines may consume storage and resources

---

## 2. Stages

Stages are the building blocks of a pipeline that execute sequentially. They define the workflow progression and control execution flow.

### Stage Execution

| Property | Behavior |
|----------|----------|
| **Sequential** | Stages run one after another; next stage waits for current stage completion |
| **Parallel Jobs** | Multiple jobs in the same stage execute simultaneously |
| **Failure Impact** | If any job fails, entire stage fails; pipeline stops (unless configured) |
| **Dependency** | Next stage only runs if all jobs in current stage succeed |
| **No Parallel Stages** | Cannot have parallel stages—only sequential execution |

### Stage Failure Handling

- If **ANY** job in a stage fails, the entire stage is marked as failed
- By default, pipeline stops and subsequent stages do not run
- Can be configured to allow failures using `allow_failure: true`
- Allows for optional non-blocking jobs (e.g., security warnings that don't halt deployment)

### Example Stage Configuration

```yaml
stages:
  - build
  - test
  - deploy

build_job:
  stage: build
  script:
    - echo 'Building...'

test_job:
  stage: test
  script:
    - echo 'Testing...'
  
deploy_job:
  stage: deploy
  script:
    - echo 'Deploying...'
```

**Execution Order:**
1. `build_job` runs first (only job in build stage)
2. `test_job` starts after build_job completes
3. `deploy_job` starts after test_job completes

---

## 3. Jobs

Jobs are the individual units of work within a stage. They run on runners and are isolated from each other.

### Job Isolation

**Critical Concept:** Jobs are self-contained and do NOT share resources

- Each job runs in its own isolated environment
- Variables and artifacts defined in one job are **NOT** accessible to other jobs by default
- Data must be explicitly shared through:
  - **Artifacts**: Files generated by jobs (for later stages)
  - **Variables**: Environment variables passed between jobs
  - **Scripts**: Explicit passing of values

**Example of Isolation:**
```yaml
job_1:
  stage: build
  script:
    - export BUILD_VAR="123"  # Only in this job

job_2:
  stage: test
  script:
    - echo $BUILD_VAR  # WILL NOT print "123"—not accessible!
```

### Job Parallelism

- Multiple jobs within the **SAME** stage run in parallel
- Jobs in different stages run sequentially
- Parallel execution speed is limited by available runners
- Example: 3 test jobs in the test stage all run simultaneously

### Example Job Configuration

```yaml
test_unit:
  stage: test
  script:
    - npm test
  allow_failure: false  # Pipeline stops if this job fails

test_integration:
  stage: test
  script:
    - npm run test:integration
  allow_failure: true  # Pipeline continues even if this fails
```

**Execution:** Both jobs run in parallel. If `test_unit` fails, the pipeline stops. If `test_integration` fails, pipeline continues.

---

## 4. Artifacts

Artifacts are files generated by jobs that can be downloaded or used by other jobs in later stages. They enable data sharing between jobs.

### Artifact Characteristics

- Generated by jobs and stored in GitLab
- Can be downloaded from the pipeline UI
- Automatically passed to jobs in later stages
- Retention period is configurable
- Typical use cases: compiled binaries, test reports, build outputs

### Artifact Retention

- **Default:** stored until pipeline is deleted
- Can be configured with `expire_in`
- Old artifacts are automatically cleaned up based on expiration
- **Common retention periods:**
  - `1 week` - short-lived build artifacts
  - `30 days` - test reports
  - `never` - important releases (use cautiously)

### How Artifacts Flow Between Stages

```
Stage 1: build_job generates dist/ → Saved as artifact
  ↓
Stage 2: test_job receives dist/ automatically
  ↓
Stage 3: deploy_job receives dist/ automatically
```

### Example Artifact Configuration

```yaml
build:
  stage: build
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
      - build/
    expire_in: 1 week

deploy:
  stage: deploy
  script:
    - ./scripts/deploy.sh
  dependencies:
    - build  # Only depend on build artifacts
```

---

## 5. Variables

Variables allow you to define dynamic values that can be used throughout pipelines. They support sensitive data management through protection mechanisms.

### Variable Types

| Type | Visibility | Use Case |
|------|-----------|----------|
| **Unprotected** | All branches | Non-sensitive config (API URLs, app names) |
| **Protected** | Protected branches only | Secrets, tokens, API keys, deployment credentials |

### Protected Variables

Protected variables are **only accessible** in pipelines for protected branches:
- Typical use: deployment credentials, API tokens, secrets
- Restrict merge permissions to users with secret access only
- Prevents accidental exposure in feature branch pipelines
- Essential for security hardening

### Variable Scopes

- **Project-level:** accessible to all pipelines in a project
- **Group-level:** accessible to pipelines in all projects within a group
- **Instance-level** (Admin only): accessible across all projects

### Variable Accessibility

Protected variables are **accessible ONLY on protected branches:**
```
Protected Branch (main) → Can access protected variables ✓
Feature Branch → Cannot access protected variables ✗
Pull Request → Cannot access protected variables ✗
```

### Example Variable Configuration

```yaml
variables:
  APP_ENV: production
  LOG_LEVEL: debug
  REGISTRY_URL: registry.example.com

deploy:
  stage: deploy
  script:
    - echo "Deploying to $APP_ENV"
    - docker push $REGISTRY_URL/myapp:latest
  only:
    - main  # Only run on main branch
```

---

## 6. Runners

Runners are machines that execute jobs defined in pipelines. They can be shared (group/instance) or specific (project). Protected runners provide additional security for sensitive operations.

### Runner Types

| Type | Scope | Best For |
|------|-------|----------|
| **Instance** | All projects | General workloads, any project |
| **Group** | All projects in group | Team-shared workloads, shared infrastructure |
| **Project** | Single project only | Specific project requirements, isolated builds |

### Runner Tags

Runners are matched to jobs using tags:
- Jobs specify required tags (e.g., `tags: [docker, linux]`)
- Runners have assigned tags (e.g., `linux`, `docker`, `python`)
- A job matches a runner if all job tags are present on the runner

**Important:** If no runner matches a job's tags, the job waits indefinitely.

### Protected Runners

Protected runners provide critical security controls:

**Behavior:**
- Can **ONLY** execute jobs on protected branches
- Prevents untrusted code from executing on protected infrastructure
- Protects deployment keys and credentials from being unintentionally accessed

**Use Case:**
```yaml
deploy_production:
  stage: deploy
  script:
    - ./scripts/deploy-prod.sh
  tags:
    - protected-runner  # Must run on protected runner
  only:
    - main  # Only on main branch
```

### Example Runner Configuration

```yaml
deploy_production:
  stage: deploy
  script:
    - ./scripts/deploy-prod.sh
  tags:
    - protected-runner
  only:
    - main

test_unit:
  stage: test
  script:
    - npm test
  tags:
    - docker  # Can run on any docker runner
```

---

## 7. Roles & Permissions

Roles define access levels and permissions within a GitLab project. Different roles have different capabilities for managing pipelines, variables, and deployment.

### Role Hierarchy (from most to least privileged)

| Owner | Maintainer | Developer | Reporter | Guest |
|-------|-----------|-----------|----------|-------|
| Full project control | Manage all aspects | Run & debug jobs | View jobs & logs | View only |

### Pipeline-Specific Permissions

- **Guest:** Can view public pipelines
- **Reporter:** Can view pipelines and job logs
- **Developer:** Can view, run, and debug pipelines; download artifacts; retry jobs
- **Maintainer:** Can manage pipeline settings, protected variables, deploy keys, runner configuration
- **Owner:** Full control; can delete and manage all project aspects

### Critical Security Note

**For protected variables and protected runners:**
- Only assign **Maintainer** or **Owner** roles to users who need to access secrets
- Restrict merge permissions to protected branches to users with appropriate roles
- Prevents unauthorized secret exposure in unprotected branches

---

## 8. What Pipelines CAN Do ✓

### Build & Compilation
- Compile source code, generate binaries, bundle assets
- Create Docker images and push to registries
- Generate executable artifacts for deployment

### Testing
- Run unit tests, integration tests, performance tests, security scans
- Generate and track test reports and coverage metrics
- Analyze code quality and lint violations

### Deployment
- Deploy applications to staging, production, or cloud environments
- Execute database migrations and infrastructure setup
- Support blue-green, canary, and rolling deployments

### Data Sharing
- Share data between jobs using artifacts
- Pass environment variables across stages
- Store and retrieve build outputs for consumption by later stages

### Automation
- Run scheduled pipelines on a cron basis
- Trigger manual pipeline runs via UI
- Trigger child pipelines or downstream projects
- Execute workflows based on Git events or webhooks

### Security
- Store and manage sensitive data with protected variables
- Restrict deployment access to protected branches
- Use protected runners to prevent untrusted code execution
- Implement role-based access control (RBAC)

### Notifications & Reporting
- Send pipeline status notifications
- Generate test reports and coverage metrics
- Track pipeline history and performance
- Integrate with external systems (Slack, email, webhooks)

---

## 9. What Pipelines CANNOT Do ✗

### Resource Constraints
- Cannot exceed runner hardware limits (CPU, RAM, disk)
- Cannot overwrite or extend available runner resources dynamically
- Cannot automatically scale runners (manual or external scaling required)
- Limited by runner capacity and availability

### Job Isolation
- Cannot directly access variables or data from other jobs without artifacts
- Cannot share filesystem or memory between parallel jobs
- Each job has isolated environment—no shared state
- Data sharing requires explicit mechanisms (artifacts, variables)

### Pipeline Structure
- **Cannot have parallel stages** (stages must run sequentially)
- Cannot skip intermediate stages while maintaining dependencies
- Cannot dynamically modify stage order at runtime
- Cannot reorder stages based on conditions

### Artifact Limitations
- Cannot automatically merge artifacts from multiple jobs
- Artifacts are removed after expiration and **cannot be recovered**
- Cannot reference artifacts from different pipelines
- Limited artifact storage by GitLab instance configuration

### Deletion Behavior
- Deleting a parent pipeline does **NOT** delete child pipelines
- Must manually manage orphaned child pipelines
- Orphaned pipelines consume storage and may cause confusion

### Runner Assignment
- Cannot force a job to run on a specific runner if tags don't match
- Jobs will wait indefinitely if no runner with matching tags is available
- Cannot override runner selection at runtime
- No fallback mechanism if primary runner is unavailable

### Variable Accessibility
- Protected variables are **ONLY** accessible on protected branches
- Cannot access protected variables from unprotected branches or pull requests
- Cannot dynamically set protected variables from job scripts
- Cannot inherit protected variables from parent groups in certain cases

### Monitoring & Debugging
- Cannot execute arbitrary code outside of defined jobs
- Cannot directly modify pipeline configuration at runtime (requires new commit)
- Limited debugging capabilities; relies on job logs and output
- Cannot inspect runner internals or middleware

### Parallel Execution
- Cannot have jobs run in parallel across different stages
- No built-in mechanism for inter-stage parallelism
- Performance limited by sequential stage execution

---

## 10. Best Practices & Key Takeaways

### Pipeline Design

1. **Keep stages focused and independent**
   - `build` → `test` → `deploy` structure
   - Each stage has single responsibility
   - Avoid bloated stages with many jobs

2. **Use meaningful stage names**
   - Clear intent (build, test, staging, production)
   - Matches typical CI/CD workflow
   - Easy to understand and troubleshoot

3. **Parallelize jobs within stages to improve performance**
   - Multiple test jobs in parallel
   - Reduce overall pipeline execution time
   - Balance parallelism with runner availability

### Data Sharing

1. **Use artifacts explicitly for passing data between stages**
   ```yaml
   artifacts:
     paths:
       - build/
       - dist/
   ```

2. **Document artifact dependencies** with `dependencies`
   ```yaml
   dependencies:
     - build  # Only this job's artifacts
   ```

3. **Set appropriate artifact expiration** to save storage
   - Short-lived builds: `expire_in: 1 day`
   - Test reports: `expire_in: 30 days`
   - Releases: `expire_in: never`

### Security

1. **Always use protected variables for secrets**
   - Never hardcode in `.gitlab-ci.yml`
   - Use GitLab's variable management UI
   - Mark as protected for branch-level access control

2. **Restrict merge permissions** to protect sensitive branches
   - Only authorized users merge to main
   - Prevents unauthorized deployments
   - Integrates with protected variables

3. **Tag deployment jobs to use protected runners**
   ```yaml
   tags:
     - protected-runner
   ```

4. **Never hardcode secrets** in pipeline files
   - Use variables, secrets management systems
   - Rotate credentials regularly
   - Audit secret access

### Runner Management

1. **Use consistent tagging** for runner selection
   - Standardize tag naming (`docker`, `linux`, `python`)
   - Document runner capabilities
   - Avoid ambiguous or overlapping tags

2. **Monitor runner availability and performance**
   - Check runner health regularly
   - Track job wait times
   - Scale runners based on demand

3. **Ensure job timeout settings** match expected execution time
   - Prevent orphaned jobs
   - Catch hanging processes
   - Set reasonable defaults per stage

### Error Handling

1. **Use `allow_failure: true`** for non-critical jobs
   ```yaml
   lint:
     stage: test
     allow_failure: true  # Don't block deployment
   ```

2. **Configure failure policies** in job configuration
   - Define what counts as failure
   - Handle edge cases gracefully
   - Provide meaningful error messages

3. **Test failure scenarios** in non-production branches first
   - Validate error handling
   - Ensure recovery procedures work
   - Document failure recovery steps

---

## 11. Quick Reference: Common Scenarios

### Scenario 1: Multi-Stage Build & Deploy Pipeline

```yaml
stages:
  - build
  - test
  - deploy

build:
  stage: build
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 week

test_unit:
  stage: test
  script:
    - npm test

test_integration:
  stage: test
  script:
    - npm run test:integration
  allow_failure: true

deploy_staging:
  stage: deploy
  script:
    - ./scripts/deploy.sh staging
  only:
    - develop

deploy_production:
  stage: deploy
  script:
    - ./scripts/deploy.sh production
  only:
    - main
  tags:
    - protected-runner
```

**Execution Flow:**
1. `build` runs and generates `dist/` artifact
2. `test_unit` and `test_integration` run in parallel
3. If both tests pass (or integration fails but allowed), deploy stages run
4. `deploy_staging` runs only from `develop` branch
5. `deploy_production` runs only from `main` branch with protected runner

### Scenario 2: Using Protected Variables for Deployment

```yaml
variables:
  REGISTRY: registry.example.com
  APP_NAME: myapp

deploy_production:
  stage: deploy
  script:
    - docker login -u $DOCKER_USER -p $DOCKER_PASS $REGISTRY
    - docker push $REGISTRY/$APP_NAME:latest
  only:
    - main
  variables:
    DOCKER_USER: $PROD_DOCKER_USER  # Protected variable
    DOCKER_PASS: $PROD_DOCKER_PASS  # Protected variable
```

**Security Notes:**
- Protected variables only accessible on `main` branch
- Feature branches cannot access `$PROD_DOCKER_USER` or `$PROD_DOCKER_PASS`
- Only authorized maintainers can view/edit these variables

### Scenario 3: Conditional Job Execution

```yaml
run_tests:
  stage: test
  script:
    - npm test
  except:
    - tags

deploy:
  stage: deploy
  script:
    - ./scripts/deploy.sh
  only:
    - main
    - tags
```

**Behavior:**
- `run_tests` skips if push is for a tag
- `deploy` only runs on `main` branch or tag pushes

---

## Summary: Key Concepts

| Concept | Key Point |
|---------|-----------|
| **Pipeline** | Automated workflow with sequential stages |
| **Stages** | Run sequentially; contain parallel jobs |
| **Jobs** | Isolated units; share data via artifacts/variables |
| **Artifacts** | Files passed between stages automatically |
| **Variables** | Configuration; protected ones for secrets |
| **Runners** | Execute jobs; matched via tags |
| **Roles** | Define permissions (Guest → Owner) |

---

## Troubleshooting Guide

### Problem: Job Waits Indefinitely
**Cause:** No runner matches job tags
**Solution:** Verify runner tags match job tags in `.gitlab-ci.yml`

### Problem: Cannot Access Protected Variable
**Cause:** Not on protected branch or insufficient permissions
**Solution:** Ensure on protected branch; check user role

### Problem: Artifacts Not Available in Next Stage
**Cause:** Artifact expiration or missing from configuration
**Solution:** Check artifact definition and expiration settings

### Problem: Pipeline Stops on First Job Failure
**Cause:** Missing `allow_failure: true`
**Solution:** Add `allow_failure: true` for non-critical jobs

### Problem: Cannot See Child Pipeline Output
**Cause:** Child pipeline deleted separately; parent pipeline unaware
**Solution:** Maintain parent-child pipeline lifecycle; document relationship

---

**Last Updated:** 2026-03-05  
**GitLab Version:** 15.0+
