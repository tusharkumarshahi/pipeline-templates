# Pipeline Template Design Document

## Overview

This document explains the architecture, design decisions, and onboarding process for the reusable CI/CD pipeline template.

## Architecture

### Component Diagram
```
┌─────────────────────────────────────┐
│   Pipeline Template Repository      │
│                                     │
│  ┌───────────────────────────────┐ │
│  │ Reusable Workflow             │ │
│  │ - Input parameters            │ │
│  │ - Build logic                 │ │
│  │ - Docker build & push         │ │
│  └───────────────────────────────┘ │
└─────────────────────────────────────┘
           ▲         ▲         ▲
           │         │         │
           │         │         │
    ┌──────┴───┐ ┌──┴─────┐ ┌─┴────────┐
    │ App 1    │ │ App 2  │ │ App N    │
    │ Workflow │ │ Workflow│ │ Workflow │
    └──────────┘ └────────┘ └──────────┘
```

### How It Works

1. **Template Repository (Hub)**
   - Contains the reusable workflow with all CI/CD logic
   - Accepts configurable inputs for flexibility
   - Handles multiple build tools and scenarios

2. **Consumer Repositories (Spokes)**
   - Individual application repositories
   - Contain minimal workflow files that call the template
   - Pass application-specific configuration as inputs

3. **Execution Flow**
```
   Push to App Repo
        ↓
   Trigger Consumer Workflow
        ↓
   Call Template Workflow (with inputs)
        ↓
   Template Executes:
     - Checkout app code
     - Build application
     - Build Docker image
     - Push to Docker Hub
        ↓
   Image Available on Docker Hub
```

## Design Decisions

### 1. Input Parameters

**Why configurable inputs?**
- Different applications have different needs
- Avoids code duplication
- Allows template to serve multiple use cases

**Key inputs:**
- `image_name`: Each app needs a unique Docker image name
- `build_tool`: Supports Maven, Gradle, Go, npm, or none
- `dockerfile_path`: Apps may have Dockerfiles in different locations
- `java_version`: Different apps may require different Java versions

### 2. Multi-Tool Support

**Why support multiple build tools?**
- Organizations often have polyglot architectures
- One template can serve Java, Go, Node.js, Python projects
- Reduces maintenance burden

**Implementation:**
- Conditional steps using `if:` expressions
- Only relevant steps execute based on `build_tool` input

### 3. Docker Tagging Strategy

**Automatic tags created:**
- `latest` - Always points to the most recent main branch build
- `<branch>` - Branch-specific tags (e.g., `main`, `develop`)
- `<branch>-<sha>` - Immutable tags with commit SHA
- Custom suffix - For environment-specific tags (e.g., `prod`, `staging`)

**Why multiple tags?**
- `latest` - Easy deployment in dev environments
- SHA tags - Reproducibility and rollback capability
- Branch tags - Environment-specific deployments

### 4. Secret Handling

**Why pass secrets from consumer to template?**
- Security: Secrets stay in the consuming repository
- Isolation: Each app can use different Docker Hub accounts
- Compliance: Follows GitHub's security best practices

**Implementation:**
```yaml
secrets:
  DOCKER_USERNAME:
    required: true
  DOCKER_TOKEN:
    required: true
```

### 5. Caching Strategy

**Build cache:**
- Uses Docker layer caching
- Stores cache as separate image tag (`buildcache`)
- Significantly speeds up subsequent builds

**Why important?**
- Faster feedback for developers
- Reduced GitHub Actions minutes usage
- Better CI/CD performance

## Reusability Across Repositories

### Benefits

1. **Centralized Maintenance**
   - Update once, all consuming repos benefit
   - Bug fixes propagate automatically
   - Add features in one place

2. **Consistency**
   - All applications follow same build process
   - Standardized Docker image structure
   - Uniform tagging strategy

3. **Reduced Boilerplate**
   - Consumer workflows are ~15 lines instead of ~60+
   - Less code to maintain per repository
   - Easier to onboard new applications

4. **Flexibility**
   - Input parameters allow customization
   - Can handle different project structures
   - Supports multiple programming languages

### Version Management

**Using `@main` (Rolling Updates):**
```yaml
uses: username/pipeline-template/.github/workflows/docker-build-push.yml@main
```
- Always uses latest version
- Automatic updates
- Best for: Active development, small teams

**Using Tags (Stable Versions):**
```yaml
uses: username/pipeline-template/.github/workflows/docker-build-push.yml@v1
```
- Pin to specific version
- Controlled updates
- Best for: Production, large teams, compliance requirements

## Onboarding Process

### Step-by-Step Guide for New Applications

#### Phase 1: Prerequisites (5 minutes)

1. **Prepare Application**
   - Ensure application builds successfully locally
   - Create a `Dockerfile` that works with your application
   - Test Docker build locally

2. **Gather Information**
   - Identify build tool (Maven, Gradle, Go, npm)
   - Decide on Docker image name
   - Determine Java/Go/Node version if applicable

3. **Docker Hub Setup**
   - Have Docker Hub account ready
   - Create access token at https://hub.docker.com/settings/security
   - Note down username and token

#### Phase 2: Repository Configuration (3 minutes)

1. **Add GitHub Secrets**
```
   Repository → Settings → Secrets and variables → Actions
   
   Add:
   - DOCKER_USERNAME: <your-dockerhub-username>
   - DOCKER_TOKEN: <your-dockerhub-access-token>
```

2. **Verify Dockerfile**
   - Ensure `Dockerfile` is in repository root
   - Or note custom path for workflow configuration

#### Phase 3: Workflow Creation (2 minutes)

1. **Create Workflow File**
```bash
   mkdir -p .github/workflows
```

2. **Copy Template**
   Create `.github/workflows/ci.yml`:
```yaml
   name: CI Pipeline
   
   on:
     push:
       branches: [ main ]
   
   jobs:
     build:
       uses: YOUR-USERNAME/pipeline-template/.github/workflows/docker-build-push.yml@main
       with:
         image_name: YOUR-APP-NAME  # Change this!
         build_tool: maven           # Change to your tool
       secrets:
         DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
         DOCKER_TOKEN: ${{ secrets.DOCKER_TOKEN }}
```

3. **Customize Parameters**
   - Replace `YOUR-USERNAME` with template repo owner
   - Replace `YOUR-APP-NAME` with desired image name
   - Set `build_tool` to: `maven`, `gradle`, `go`, `npm`, or `none`
   - Add optional parameters if needed

#### Phase 4: Testing (5 minutes)

1. **Commit and Push**
```bash
   git add .github/workflows/ci.yml
   git commit -m "Add CI pipeline"
   git push origin main
```

2. **Monitor Execution**
   - Go to repository Actions tab
   - Watch workflow execute
   - Verify all steps complete successfully

3. **Verify Docker Hub**
   - Check Docker Hub for new image
   - Verify tags are created correctly

#### Phase 5: Validation (2 minutes)

1. **Test Docker Image**
```bash
   docker pull YOUR-USERNAME/YOUR-APP-NAME:latest
   docker run -p 8080:8080 YOUR-USERNAME/YOUR-APP-NAME:latest
```

2. **Make Test Commit**
   - Make a small change
   - Push to main
   - Verify pipeline runs automatically

### Total Onboarding Time: ~15-20 minutes

## Advanced Configurations

### Multi-Stage Dockerfiles

For optimized images:
```dockerfile
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Custom Build Context

If your Dockerfile is in a subdirectory:
```yaml
with:
  image_name: my-app
  dockerfile_path: ./docker/Dockerfile
  build_context: ./app
  build_tool: maven
```

### Environment-Specific Tags

For staging deployments:
```yaml
with:
  image_name: my-app
  docker_tag_suffix: staging
  build_tool: maven
```

## Troubleshooting

### Common Issues

1. **"Resource not accessible by integration"**
   - Cause: Secrets not configured
   - Fix: Add DOCKER_USERNAME and DOCKER_TOKEN secrets

2. **"Dockerfile not found"**
   - Cause: Dockerfile in wrong location
   - Fix: Use `dockerfile_path` input parameter

3. **"Invalid tag format"**
   - Cause: image_name contains invalid characters
   - Fix: Use lowercase letters, numbers, hyphens only

4. **Build fails with "command not found"**
   - Cause: Wrong build_tool specified
   - Fix: Verify build_tool matches your project

## Maintenance and Updates

### Updating the Template

1. **Test Changes**
   - Create a test branch in template repo
   - Test with a consumer repo using `@test-branch`

2. **Version Release**
   - Tag stable releases: `git tag v1.0.0`
   - Update documentation with breaking changes

3. **Communication**
   - Notify teams of updates
   - Provide migration guide for breaking changes

### Monitoring

- Review GitHub Actions usage
- Check for failed workflows across consuming repos
- Gather feedback from development teams

## Future Enhancements

Potential improvements:
- Add security scanning (Trivy, Snyk)
- Implement automated testing stage
- Add Kubernetes deployment step
- Support for multiple registries (ECR, GCR, ACR)
- Add notification integrations (Slack, email)
- Implement rollback capabilities

## Conclusion

This reusable pipeline template provides:
- ✅ Centralized CI/CD logic
- ✅ Easy onboarding for new applications
- ✅ Consistency across projects
- ✅ Reduced maintenance burden
- ✅ Flexibility for different use cases

For questions or issues, please open an issue in the template repository.
