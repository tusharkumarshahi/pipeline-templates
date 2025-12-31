# Reusable CI/CD Pipeline Template

This repository contains a reusable GitHub Actions workflow for building applications and pushing Docker images to Docker Hub.

## Features

- ✅ Supports multiple build tools: Maven, Gradle, Go, npm, or none
- ✅ Builds Docker images with automatic tagging
- ✅ Pushes to Docker Hub with secure credential handling
- ✅ Configurable for different project structures
- ✅ Build caching for faster builds

## Usage

### Prerequisites

In your application repository, ensure you have:
1. A `Dockerfile` in the root (or specify custom path)
2. GitHub Secrets configured:
   - `DOCKER_USERNAME` - Your Docker Hub username
   - `DOCKER_TOKEN` - Your Docker Hub access token

### Basic Usage

Create `.github/workflows/ci.yml` in your application repository:
```yaml
name: CI Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    uses: YOUR-USERNAME/pipeline-template/.github/workflows/docker-build-push.yml@main
    with:
      image_name: my-app-name
      build_tool: maven
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_TOKEN: ${{ secrets.DOCKER_TOKEN }}
```

### Configuration Options

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `image_name` | Docker image name (without username/registry) | Yes | - |
| `dockerfile_path` | Path to Dockerfile | No | `./Dockerfile` |
| `build_context` | Docker build context | No | `.` |
| `build_tool` | Build tool: `maven`, `gradle`, `go`, `npm`, `none` | No | `maven` |
| `java_version` | Java version for Maven/Gradle | No | `17` |
| `docker_tag_suffix` | Additional tag suffix | No | `''` |

### Examples

#### Java Maven Project
```yaml
uses: YOUR-USERNAME/pipeline-template/.github/workflows/docker-build-push.yml@main
with:
  image_name: java-spring-app
  build_tool: maven
  java_version: '17'
secrets:
  DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
  DOCKER_TOKEN: ${{ secrets.DOCKER_TOKEN }}
```

#### Go Application
```yaml
uses: YOUR-USERNAME/pipeline-template/.github/workflows/docker-build-push.yml@main
with:
  image_name: go-microservice
  build_tool: go
secrets:
  DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
  DOCKER_TOKEN: ${{ secrets.DOCKER_TOKEN }}
```

#### Node.js Application
```yaml
uses: YOUR-USERNAME/pipeline-template/.github/workflows/docker-build-push.yml@main
with:
  image_name: node-api
  build_tool: npm
secrets:
  DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
  DOCKER_TOKEN: ${{ secrets.DOCKER_TOKEN }}
```

#### Custom Dockerfile Location
```yaml
uses: YOUR-USERNAME/pipeline-template/.github/workflows/docker-build-push.yml@main
with:
  image_name: custom-app
  dockerfile_path: ./docker/Dockerfile.prod
  build_context: ./app
  build_tool: maven
secrets:
  DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
  DOCKER_TOKEN: ${{ secrets.DOCKER_TOKEN }}
```

## Docker Image Tagging Strategy

The workflow automatically creates multiple tags:
- `latest` (only on main branch)
- Branch name (e.g., `main`, `develop`)
- Short commit SHA prefixed with branch (e.g., `main-abc1234`)
- Custom suffix if provided (e.g., `prod`, `staging`)

## Onboarding a New Application

Follow these steps to use this pipeline in a new repository:

### Step 1: Prepare Your Repository
Ensure your repository has:
- Application source code
- A `Dockerfile` that builds your application
- Proper build configuration (pom.xml, build.gradle, go.mod, package.json, etc.)

### Step 2: Configure Secrets
1. Go to your repository Settings → Secrets and variables → Actions
2. Add two secrets:
   - `DOCKER_USERNAME`: Your Docker Hub username
   - `DOCKER_TOKEN`: Docker Hub access token (create at hub.docker.com/settings/security)

### Step 3: Create Workflow File
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
      image_name: YOUR-APP-NAME
      build_tool: maven  # Change to your build tool
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_TOKEN: ${{ secrets.DOCKER_TOKEN }}
```

### Step 4: Commit and Push
```bash
git add .github/workflows/ci.yml
git commit -m "Add CI pipeline using reusable template"
git push origin main
```

### Step 5: Verify
Check the Actions tab in your repository to see the workflow running.

## Maintenance

To update the pipeline for all consuming repositories:
1. Make changes to this template repository
2. All repositories using `@main` will automatically use the updated version
3. For stable versions, use tags: `@v1`, `@v2`, etc.

## Support

For issues or questions, please open an issue in this repository.
