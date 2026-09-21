# Sparta App — Jenkins CI/CD Pipeline

## Overview

This repository contains the Sparta application and demonstrates a **CI/CD workflow using GitHub and Jenkins**.

The purpose of the repository is to demonstrate how changes can be developed safely on a `dev` branch, automatically built and tested by Jenkins, and then promoted to the `main` branch only when the CI job completes successfully.

The workflow separates **development and production-ready code**:

```text
Developer
    │
    │ Push changes
    ▼
  dev branch
    │
    ▼
Jenkins Job 1
CI / Build & Test
    │
    │ SUCCESS
    ▼
Jenkins Job 2
Merge dev → main
    │
    ▼
  main branch
```

This provides an automated way of verifying changes before they are merged into `main`.

---

## Repository Purpose

The repository was created to demonstrate practical knowledge of:

* Git and GitHub
* Git branching
* Jenkins
* Continuous Integration (CI)
* Continuous Delivery / Deployment concepts
* Automated builds
* Automated testing
* Jenkins job chaining
* SSH authentication
* Automated branch merging

The main objective is to show how Jenkins can automate the process of validating and promoting changes between Git branches.

---

# Branch Strategy

The repository uses two primary branches:

### `dev`

The `dev` branch is used for ongoing development.

Developers make changes and push them to this branch.

For example:

```bash
git checkout dev
git add .
git commit -m "Add new feature"
git push origin dev
```

A push to `dev` triggers the first Jenkins job.

### `main`

The `main` branch contains changes that have successfully passed the CI process.

Changes should not be pushed directly to `main` as part of the normal workflow.

Instead:

```text
dev → Jenkins CI → successful build → Jenkins merge → main
```

This means that the merge into `main` is automated only after the CI process has completed successfully.

---

# Jenkins Pipeline

The workflow uses two Jenkins jobs.

## Job 1 — CI / Build & Test

**Jenkins job:**

```text
kacper-spapp-job1-ci-test
```

The purpose of Job 1 is to verify that the application can successfully build and pass its tests.

### Trigger

Job 1 is triggered when changes are pushed to the GitHub repository.

The workflow starts with:

```text
GitHub push
     ↓
dev branch
     ↓
Jenkins Job 1
```

### What Job 1 does

The job:

1. Checks out the repository.
2. Retrieves the latest changes from `dev`.
3. Builds the application.
4. Runs the automated tests.
5. Reports the result to Jenkins.

If the build or tests fail, the job is marked as failed.

If everything succeeds, the job is marked as successful.

---

# Jenkins Job 2 — Merge `dev` into `main`

**Jenkins job:**

```text
kacper-spapp-job2-ci-merge
```

Job 2 is responsible for promoting the successfully tested changes from `dev` into `main`.

It is triggered **after Job 1 completes successfully**.

The relationship between the jobs is:

```text
kacper-spapp-job1-ci-test
              │
              │ SUCCESS
              ▼
kacper-spapp-job2-ci-merge
```

If Job 1 fails, Job 2 does not run.

---

## How Job 2 Works

Job 2 uses Git commands to update `main` and merge the latest version of `dev`.

The process is:



```bash
git checkout main
```

Switches the Jenkins workspace to the `main` branch.

Next:

```bash
git pull origin main
```

Updates the local `main` branch with the latest version from GitHub.

Then:

```bash
git merge origin/dev --no-edit
```

Merges the latest remote `dev` branch into `main`.

Finally:

```bash
git push origin main
```

Pushes the updated `main` branch back to GitHub.

The complete process is therefore:

```text
             GitHub
                │
          origin/dev
                │
                ▼
         git fetch origin
                │
                ▼
         checkout main
                │
                ▼
         pull latest main
                │
                ▼
       merge origin/dev
                │
                ▼
         push origin main
                │
                ▼
             GitHub
             main
```
---
# Jenkins Job Setup

## Job 1 — CI / Build & Test

**Job name:**

```text
kacper-spapp-job1-ci-test
```

### 1. Create the Jenkins job

Create a new Jenkins **Freestyle project** named:

```text
kacper-spapp-job1-ci-test
```

### 2. Configure Source Code Management

Under **Source Code Management**, select **Git**.

Repository URL:

```text
git@github.com:kf304/pre-tech613-sparta-app-cicd-jenkins.git
```

Set the branch to:

```text
*/dev
```

Select the Jenkins SSH credential:

```text
kacper_github_key
```

This allows Jenkins to securely clone the repository and access the `dev` branch.

### 3. Configure the build trigger

Under **Build Triggers**, enable:

```text
GitHub hook trigger for GITScm polling
```

This allows the GitHub webhook to trigger Job 1 when a push is made.

### 4. Configure the build step

Under **Build Steps**, add **Execute shell**.

The build script is:

```bash
cd app
npm install
npm test
```

This installs the application's dependencies and runs the automated tests.

If the tests pass, Jenkins marks Job 1 as successful.

---

## Job 2 — Merge `dev` into `main`

**Job name:**

```text
kacper-spapp-job2-ci-merge
```

### 1. Create the Jenkins job

Create a second Jenkins **Freestyle project** named:

```text
kacper-spapp-job2-ci-merge
```

### 2. Configure Source Code Management

Under **Source Code Management**, select **Git**.

Repository URL:

```text
git@github.com:kf304/pre-tech613-sparta-app-cicd-jenkins.git
```

Select:

```text
kacper-jenkins
```

as the Git credential.

The job is configured to work with the repository branches and merge the latest `dev` changes into `main`.

### 3. Configure the upstream trigger

Under **Build Triggers**, configure Job 2 to build after:

```text
kacper-spapp-job1-ci-test
```

Set the trigger so that Job 2 only runs when Job 1 is successful.

This creates the dependency:

```text
Job 1
  │
  │ SUCCESS
  ▼
Job 2
```

If Job 1 fails, Job 2 is not triggered.

### 4. Configure SSH Agent

Enable the Jenkins **SSH Agent** build environment and select:

```text
kacper-jenkins
```

This makes the SSH key available to Git commands executed by the shell.

This is required because Job 2 needs to authenticate with GitHub when running:

```bash
git pull origin main
git push origin main
```

The private SSH key is stored in Jenkins credentials rather than being included in the repository or shell script.

### 5. Configure the merge build step

Under **Build Steps**, add **Execute shell**:

```bash
git checkout main
git pull origin main
git merge origin/dev --no-edit
git push origin main
```

The commands:

1. Switch to `main`.
2. Update `main` from GitHub.
3. Merge the latest `dev` branch into `main`.
4. Push the updated `main` branch back to GitHub.
---

# GitHub Webhook Setup

A GitHub webhook is used to automatically notify Jenkins when changes are pushed to the repository.

## Jenkins configuration

In Job 1:

**Configure → Build Triggers**

Enable:

```text
GitHub hook trigger for GITScm polling
```

This allows Jenkins to respond to GitHub webhook events.

## GitHub configuration

In the GitHub repository, navigate to:

```text
Settings
→ Webhooks
→ Add webhook
```

Configure the webhook with the Jenkins GitHub webhook endpoint:

```text
<YOUR-JENKINS-URL>/github-webhook/
```

Set:

```text
Content type: application/json
```

Select:

```text
Just the push event
```

Ensure the webhook is:

```text
Active
```

The resulting workflow is:

```text
Developer pushes to dev
        │
        ▼
      GitHub
        │
        │ Webhook
        ▼
     Jenkins
        │
        ▼
     Job 1
```

This removes the need to manually start Job 1 after every change to `dev`.

---

# Jenkins Authentication

Jenkins needs permission to access the GitHub repository.

An SSH credential is configured in Jenkins:

```text
kacper-jenkins
```

This credential provides read/write access to the repository.

The repository uses an SSH remote:

```text
git@github.com:kf304/pre-tech613-sparta-app-cicd-jenkins.git
```

The Jenkins Git plugin uses the configured credential when checking out the repository.

For Git commands executed manually inside a shell step, the SSH credential also needs to be made available to the shell using the Jenkins **SSH Agent** configuration.

This allows commands such as:

```bash
git pull origin main
git push origin main
```

to authenticate with GitHub.

---

# Complete CI/CD Workflow

The complete workflow is:

### 1. Developer makes a change

Changes are made locally on the `dev` branch.

```bash
git checkout dev
```

### 2. Changes are committed

```bash
git add .
git commit -m "Add application change"
```

### 3. Changes are pushed

```bash
git push origin dev
```

### 4. GitHub triggers Jenkins

The GitHub push triggers:

```text
kacper-spapp-job1-ci-test
```

### 5. Jenkins builds and tests

Job 1 checks out `dev`, builds the application and runs the tests.

```text
Build + Tests
     │
     ├── FAIL → Stop
     │
     └── SUCCESS
             │
             ▼
```

### 6. Job 2 is triggered

A successful Job 1 automatically triggers:

```text
kacper-spapp-job2-ci-merge
```

### 7. `dev` is merged into `main`

Job 2 updates `main` and merges the tested changes.

```text
dev
 │
 │ merge
 ▼
main
```

### 8. Updated `main` is pushed to GitHub

```bash
git push origin main
```

The GitHub repository now contains the successfully tested changes on `main`.

---

# Why Use Two Jobs?

Separating the workflow into two Jenkins jobs provides a clear distinction between **validation** and **promotion**.

### Job 1 — Validate

The first job answers:

> "Does this change build and pass the tests?"

### Job 2 — Promote

The second job answers:

> "The change has passed CI. Can it now be merged into `main`?"

This prevents a failed build or test run from automatically being promoted to `main`.

```text
                 DEVELOPMENT
                      │
                      ▼
                    dev
                      │
                      ▼
             ┌─────────────────┐
             │    Jenkins      │
             │      Job 1      │
             │                 │
             │ Build + Test    │
             └────────┬────────┘
                      │
                ┌─────┴─────┐
                │           │
              FAIL        SUCCESS
                │           │
                ▼           ▼
              STOP       Job 2
                           │
                           ▼
                     Merge dev → main
                           │
                           ▼
                          main
```

---

# Benefits of the Workflow

This setup demonstrates several important CI/CD principles.

### Automated validation

Every push to `dev` can automatically trigger a build and test process.

### Reduced manual work

Developers do not need to manually build and test the application before every promotion.

### Controlled promotion

Only changes that successfully pass Job 1 are promoted by Job 2.

### Separation of environments

The `dev` branch provides a development area while `main` represents the successfully validated code.

### Reproducibility

The same Jenkins process can be run consistently rather than relying on a developer remembering every step manually.
