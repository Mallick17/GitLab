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
