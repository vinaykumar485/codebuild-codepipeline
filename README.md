# codebuild-codepipeline

# AWS CodeBuild — How CodePipeline, Service Quotas & buildspec.yml Work Together

🚀 **AWS CodeBuild**

AWS CodeBuild is a fully managed build service that compiles source code, runs tests, performs code analysis, builds Docker images, and produces build artifacts.

The basic relationship is:

**CodePipeline → CodeBuild → buildspec.yml**

Think of them like this:

### 1️⃣ CodePipeline = The Coordinator

CodePipeline controls the overall CI/CD workflow.

For example:

```text
GitHub
   ↓
CodePipeline
   ↓
Source Stage
   ↓
Build Stage
   ↓
Deploy Stage
```

Its job is to coordinate the workflow:

> New code arrived → get the source → trigger the build → if successful, continue to the next stage.

CodePipeline itself does not contain the Maven or Docker commands used to build the application.

---

### 2️⃣ CodeBuild = The Build Service

CodeBuild is responsible for actually executing the build.

A build can perform tasks such as:

```text
Download source
      ↓
Use Java / Maven
      ↓
mvn clean package
      ↓
Run tests
      ↓
Build Docker image
      ↓
Push image to ECR
```

You don't need to create an EC2 server specifically for CodeBuild.

AWS creates a managed build environment for the build and executes the required commands inside it.

---

### 3️⃣ Service Quota = Capacity Gate

When CodePipeline triggers CodeBuild, CodeBuild requests AWS to start the build.

AWS checks whether the account has available CodeBuild capacity under the applicable service quota.

```text
CodePipeline
     ↓
"Start Build"
     ↓
CodeBuild
     ↓
AWS checks Service Quota
     │
     ├── Quota available → ✅ Start build
     │
     └── Quota unavailable → ❌ Build rejected
```

If the quota allows the build, CodeBuild starts the build environment.

If the quota does not allow it, the build can fail **before the commands in `buildspec.yml` are executed**.

This is exactly where an `AccountLimitExceededException` can occur.

---

### 4️⃣ buildspec.yml = The Instructions

Once CodeBuild is allowed to start, it needs to know:

> **"What exactly should I do?"**

That's the purpose of `buildspec.yml`.

For example:

```yaml
phases:
  build:
    commands:
      - mvn clean package
      - docker build -t myapp .
      - docker push <image>
```

So the relationship is:

```text
CodePipeline
     │
     │ "Start the build"
     ▼
CodeBuild
     │
     │ "What should I execute?"
     ▼
buildspec.yml
     │
     ├── Maven
     ├── Docker build
     └── Docker push
```

---

## 🔥 Complete Workflow

```text
                         GitHub
                           │
                           ▼
                    ┌──────────────┐
                    │ CODEPIPELINE │
                    │              │
                    │ Coordinator  │
                    └──────┬───────┘
                           │
                           │ Start Build
                           ▼
                    ┌──────────────┐
                    │  CODEBUILD   │
                    └──────┬───────┘
                           │
                           ▼
                  ┌──────────────────┐
                  │  SERVICE QUOTA   │
                  │                  │
                  │ Is build capacity│
                  │    available?    │
                  └────────┬─────────┘
                           │
                    ┌──────┴──────┐
                    │             │
                   YES            NO
                    │             │
                    ▼             ▼
            Build Environment   ❌ Error
                    │
                    ▼
              buildspec.yml
                    │
             ┌──────┼──────┐
             ▼      ▼      ▼
           Maven  Docker  Tests
                    │
                    ▼
                   ECR
```

### 🧠 Easy way to remember

**CodePipeline** → *Controls the workflow*

**CodeBuild** → *Executes the build*

**Service Quota** → *Controls available build capacity*

**buildspec.yml** → *Defines the commands to execute*

This is the basic foundation of understanding **AWS CodePipeline + CodeBuild** before moving into the actual configuration.

