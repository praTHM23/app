# 🎯 Jenkins CI/CD Pipeline Implementation Plan

**Project**: Spring Boot Chat Application - Jenkins Docker Pipeline
**Date**: 2026-01-29
**Version**: 1.0

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Artifacts & Versioning](#artifacts--versioning)
4. [Pipeline 1: Maven Build](#pipeline-1-maven-build)
5. [Pipeline 2: Docker Build](#pipeline-2-docker-build)
6. [Dockerfile Design](#dockerfile-design)
7. [Entrypoint Script](#entrypoint-script)
8. [Jenkins Credentials](#jenkins-credentials)
9. [Execution Flow](#execution-flow)
10. [Implementation Checklist](#implementation-checklist)
11. [Files to Create](#files-to-create)

---

## 🎯 Overview

### Objective
Create a two-pipeline Jenkins CI/CD system that:
1. Builds a Maven Spring Boot application and uploads artifacts to Artifactory
2. Builds a Docker image (downloading JAR from Artifactory) and pushes to Docker registry

### Key Requirements
- ✅ Separate Maven build and Docker build pipelines
- ✅ Minimal docker-context.zip (no source code, no JAR)
- ✅ JAR downloaded from Artifactory during Docker build
- ✅ Version format: `1.0.0-YYYYMMDD`
- ✅ Auto-trigger Pipeline 2 after Pipeline 1 success
- ✅ Use UBI8 (Red Hat Universal Base Image) as base
- ✅ Secure credential handling
- ✅ Custom entrypoint script for application startup

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     PIPELINE 1: Maven Build                      │
│  ┌──────┐  ┌─────────┐  ┌──────┐  ┌─────────┐  ┌──────────┐   │
│  │Clean │→ │ Compile │→ │ Test │→ │ Package │→ │  Deploy  │   │
│  └──────┘  └─────────┘  └──────┘  └─────────┘  └──────────┘   │
│                                                                   │
│  Outputs:                                                        │
│  • app-1.0.0-20260129.jar → Maven Artifactory                   │
│  • docker-context-1.0.0-20260129.zip → Maven Artifactory        │
└─────────────────────────────────────────────────────────────────┘
                              ↓ (Auto-trigger)
┌─────────────────────────────────────────────────────────────────┐
│                   PIPELINE 2: Docker Build                       │
│  ┌──────────┐  ┌────────────┐  ┌──────────┐  ┌──────────┐     │
│  │ Download │→ │ Build Image│→ │   Tag    │→ │   Push   │     │
│  │ Context  │  │ (Download  │  │  Image   │  │  to      │     │
│  │   ZIP    │  │  JAR from  │  │          │  │ Registry │     │
│  └──────────┘  │ Artifactory)│  └──────────┘  └──────────┘     │
│                └────────────┘                                    │
│  Output:                                                         │
│  • chat-app:1.0.0-20260129 → Docker Artifactory                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📦 Artifacts & Versioning

### Version Format
**Pattern**: `BASE_VERSION-BUILD_LABEL`
- **BASE_VERSION**: `1.0.0` (configurable parameter)
- **BUILD_LABEL**: `YYYYMMDD` (auto-generated from current date)
- **Example**: `1.0.0-20260129`

### Artifacts Produced

| Artifact | Name Pattern | Destination |
|----------|-------------|-------------|
| **JAR File** | `app-1.0.0-20260129.jar` | Maven Artifactory |
| **Docker Context ZIP** | `docker-context-1.0.0-20260129.zip` | Maven Artifactory |
| **Docker Image** | `chat-app:1.0.0-20260129` | Docker Artifactory |
| **Docker Image (latest)** | `chat-app:latest` | Docker Artifactory |

### Artifactory Locations

#### Maven Repository
```
Base URL: https://na.artifactory.swg-devops.com/artifactory/ip-devops-team-sandbox-maven-local/

Full Path:
https://na.artifactory.swg-devops.com/artifactory/ip-devops-team-sandbox-maven-local/hello-world/com/chat/app/1.0.0-20260129/
├── app-1.0.0-20260129.jar
└── docker-context-1.0.0-20260129.zip
```

#### Docker Registry
```
Base URL: https://na.artifactory.swg-devops.com/artifactory/ip-devops-team-sandbox-dev-docker-local/

Images:
├── chat-app:1.0.0-20260129
└── chat-app:latest
```

---

## 🔧 Pipeline 1: Maven Build

### Purpose
Build the Spring Boot application, run tests, create artifacts, and upload to Artifactory.

### Input Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `BASE_VERSION` | String | `1.0.0` | Base version for the artifact |
| `ARTIFACTORY_MAVEN_URL` | String | `na.artifactory.swg-devops.com/...` | Maven repository URL |
| `DOCKER_PIPELINE_NAME` | String | `docker-build-pipeline` | Name of Docker pipeline to trigger |
| `TRIGGER_DOCKER_BUILD` | Boolean | `true` | Auto-trigger Docker pipeline |

### Environment Variables

| Variable | Source | Example |
|----------|--------|---------|
| `BUILD_LABEL` | Generated | `20260129` |
| `FULL_VERSION` | Computed | `1.0.0-20260129` |
| `ARTIFACT_ID` | Static | `app` |
| `GROUP_ID` | Static | `com.chat` |
| `ARTIFACTORY_CREDS` | Jenkins Credential | (username/password) |

### Stages

#### 1. Setup & Version
**Purpose**: Generate version labels and update POM
```groovy
- Generate BUILD_LABEL from current date (YYYYMMDD)
- Compute FULL_VERSION = BASE_VERSION-BUILD_LABEL
- Update pom.xml version using maven-versions-plugin
- Display build information
```

#### 2. Clean
**Purpose**: Remove old build artifacts
```bash
./mvnw clean
```

#### 3. Compile
**Purpose**: Compile Java source code
```bash
./mvnw compile
```

#### 4. Test
**Purpose**: Run unit tests and publish results
```bash
./mvnw test
# Publish JUnit test results
```

#### 5. Package
**Purpose**: Create executable JAR file
```bash
./mvnw package -DskipTests
# Verify JAR creation
# Rename to include full version
```

#### 6. Create Docker Context
**Purpose**: Create minimal docker-context.zip
```bash
# Create directory structure
mkdir -p docker-context
cp Dockerfile docker-context/
cp .dockerignore docker-context/
cp entrypoint.sh docker-context/

# Create ZIP (NO source code, NO JAR)
cd docker-context
zip -r ../docker-context-${FULL_VERSION}.zip .

# Verify contents
unzip -l docker-context-${FULL_VERSION}.zip
```

**Contents of docker-context.zip**:
- ✅ `Dockerfile`
- ✅ `.dockerignore`
- ✅ `entrypoint.sh`
- ❌ NO source code
- ❌ NO JAR file
- ❌ NO pom.xml

#### 7. Deploy to Artifactory
**Purpose**: Upload JAR and docker-context.zip to Artifactory
```bash
# Upload JAR
curl -u ${USER}:${PASS} -T app-${FULL_VERSION}.jar \
  ${ARTIFACTORY_URL}/hello-world/com/chat/app/${FULL_VERSION}/app-${FULL_VERSION}.jar

# Upload docker-context.zip
curl -u ${USER}:${PASS} -T docker-context-${FULL_VERSION}.zip \
  ${ARTIFACTORY_URL}/hello-world/com/chat/app/${FULL_VERSION}/docker-context-${FULL_VERSION}.zip
```

#### 8. Trigger Docker Build
**Purpose**: Automatically start Pipeline 2
```groovy
build job: 'docker-build-pipeline',
    parameters: [
        string(name: 'FULL_VERSION', value: env.FULL_VERSION),
        string(name: 'BUILD_LABEL', value: env.BUILD_LABEL),
        string(name: 'ARTIFACT_ID', value: env.ARTIFACT_ID),
        string(name: 'GROUP_ID', value: env.GROUP_ID)
    ],
    wait: false
```

### Success Criteria
- ✅ All tests pass
- ✅ JAR file created successfully
- ✅ docker-context.zip created with Dockerfile and entrypoint.sh
- ✅ Both artifacts uploaded to Artifactory
- ✅ Pipeline 2 triggered (if enabled)

---

## 🐳 Pipeline 2: Docker Build

### Purpose
Download docker-context.zip, build Docker image (downloading JAR from Artifactory), and push to registry.

### Input Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `FULL_VERSION` | String | (required) | Full version from Pipeline 1 |
| `BUILD_LABEL` | String | (required) | Build label (YYYYMMDD) |
| `ARTIFACT_ID` | String | `app` | Maven artifact ID |
| `GROUP_ID` | String | `com.chat` | Maven group ID |
| `ARTIFACTORY_MAVEN_URL` | String | (provided) | Maven repository URL |
| `ARTIFACTORY_DOCKER_URL` | String | (provided) | Docker registry URL |
| `IMAGE_NAME` | String | `chat-app` | Docker image name |

### Environment Variables

| Variable | Source | Example |
|----------|--------|---------|
| `ARTIFACTORY_CREDS` | Jenkins Credential | (username/password) |
| `DOCKER_REGISTRY` | Parameter | `na.artifactory.swg-devops.com/...` |
| `GIT_COMMIT_SHA` | Git | `abc123...` |
| `GIT_BRANCH` | Git | `main` |

### Stages

#### 1. Validate Parameters
**Purpose**: Ensure required parameters are provided
```groovy
if (!params.FULL_VERSION || !params.BUILD_LABEL) {
    error("Required parameters missing!")
}
```

#### 2. Download Docker Context
**Purpose**: Download docker-context.zip from Artifactory
```bash
curl -u ${USER}:${PASS} -o docker-context-${FULL_VERSION}.zip \
  ${ARTIFACTORY_URL}/hello-world/com/chat/app/${FULL_VERSION}/docker-context-${FULL_VERSION}.zip
```

#### 3. Extract Context
**Purpose**: Unzip and verify Dockerfile and entrypoint.sh
```bash
unzip -o docker-context-${FULL_VERSION}.zip
# Verify Dockerfile exists
test -f Dockerfile || exit 1
# Verify entrypoint.sh exists
test -f entrypoint.sh || exit 1
```

#### 4. Build Docker Image
**Purpose**: Multi-stage build that downloads JAR from Artifactory
```bash
docker build \
  --build-arg ARTIFACTORY_USER=${USER} \
  --build-arg ARTIFACTORY_PASSWORD=${PASS} \
  --build-arg ARTIFACTORY_MAVEN_URL=${MAVEN_URL} \
  --build-arg GROUP_ID=${GROUP_ID} \
  --build-arg ARTIFACT_ID=${ARTIFACT_ID} \
  --build-arg FULL_VERSION=${FULL_VERSION} \
  --build-arg BUILD_LABEL=${BUILD_LABEL} \
  --build-arg GIT_COMMIT_SHA=${GIT_SHA} \
  --build-arg GIT_BRANCH=${GIT_BRANCH} \
  -t ${IMAGE_NAME}:${FULL_VERSION} \
  .
```

**What happens during build**:
1. Stage 1 (downloader): Downloads JAR from Artifactory
2. Stage 2 (runtime): Copies JAR, entrypoint.sh, and creates final image

#### 5. Tag Image
**Purpose**: Tag with version and latest
```bash
# Tag with version
docker tag ${IMAGE_NAME}:${FULL_VERSION} \
  ${DOCKER_REGISTRY}/${IMAGE_NAME}:${FULL_VERSION}

# Tag as latest
docker tag ${IMAGE_NAME}:${FULL_VERSION} \
  ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
```

#### 6. Push to Registry
**Purpose**: Upload images to Artifactory Docker registry
```bash
# Login
echo ${PASS} | docker login ${DOCKER_REGISTRY} -u ${USER} --password-stdin

# Push versioned image
docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${FULL_VERSION}

# Push latest
docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest

# Logout
docker logout ${DOCKER_REGISTRY}
```

### Success Criteria
- ✅ docker-context.zip downloaded successfully
- ✅ Docker image built without errors
- ✅ JAR downloaded from Artifactory during build
- ✅ Images tagged correctly
- ✅ Images pushed to Docker registry

---

## 🐋 Dockerfile Design

### Multi-Stage Build Strategy

#### Stage 1: Downloader (UBI8)
**Purpose**: Download JAR from Artifactory
```dockerfile
FROM registry.access.redhat.com/ubi8/ubi:8.10 AS downloader

ARG ARTIFACTORY_USER
ARG ARTIFACTORY_PASSWORD
ARG ARTIFACTORY_MAVEN_URL
ARG GROUP_ID
ARG ARTIFACT_ID
ARG FULL_VERSION

# Install curl
RUN dnf update -y && dnf install -y curl && dnf clean all

WORKDIR /work

# Download JAR
RUN curl -u ${ARTIFACTORY_USER}:${ARTIFACTORY_PASSWORD} \
    -o app.jar \
    "https://${ARTIFACTORY_MAVEN_URL}/hello-world/${GROUP_ID}/${ARTIFACT_ID}/${FULL_VERSION}/${ARTIFACT_ID}-${FULL_VERSION}.jar"
```

**Key Points**:
- ✅ Uses UBI8 base image
- ✅ Credentials passed as build args (not stored in image)
- ✅ Downloads JAR from Artifactory
- ✅ This stage is discarded in final image

#### Stage 2: Runtime (UBI8)
**Purpose**: Create final runtime image
```dockerfile
FROM registry.access.redhat.com/ubi8/ubi:8.10

ARG APP_USER=spring
ARG FULL_VERSION
ARG BUILD_LABEL
ARG GIT_COMMIT_SHA
ARG GIT_BRANCH

# Install Java 17
RUN dnf update -y && \
    dnf install -y java-17-openjdk-headless && \
    dnf clean all

# Create non-root user
RUN useradd -r -u 1001 -g 0 ${APP_USER} && \
    mkdir -p /opt/app && \
    chown -R ${APP_USER}:0 /opt/app && \
    chmod -R g=u /opt/app

# Add labels for traceability
LABEL name="chat-app" \
      version="${FULL_VERSION}" \
      build-label="${BUILD_LABEL}" \
      git-commit-sha="${GIT_COMMIT_SHA}" \
      git-branch="${GIT_BRANCH}"

WORKDIR /opt/app

# Copy JAR from downloader stage
COPY --from=downloader --chown=${APP_USER}:0 /work/app.jar app.jar

# Copy entrypoint script
COPY --chown=${APP_USER}:0 entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

# Expose application port
EXPOSE 8080

# Switch to non-root user
USER ${APP_USER}

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

# Run application via entrypoint script
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
```

**Key Points**:
- ✅ Uses same UBI8 base
- ✅ Installs Java 17 JRE only (not JDK)
- ✅ Non-root user for security
- ✅ Labels for traceability
- ✅ Health check included
- ✅ Custom entrypoint script
- ✅ No credentials in final image

### Security Features
1. **Multi-stage build**: Credentials only in build stage, not in final image
2. **Non-root user**: Application runs as `spring` user (UID 1001)
3. **Minimal image**: Only JRE and application, no build tools
4. **Build args**: Credentials passed at build time, not stored
5. **Health checks**: Container health monitoring
6. **Executable permissions**: Proper file permissions for entrypoint

---

## 📜 Entrypoint Script

### Purpose
The `entrypoint.sh` script provides a flexible way to:
- Set environment variables
- Configure JVM options
- Handle signals gracefully
- Add pre-startup checks
- Enable debugging/profiling

### Script Design

```bash
#!/bin/bash
set -e

echo "=========================================="
echo "Starting Chat Application"
echo "=========================================="

# Environment variables with defaults
APP_JAR="${APP_JAR:-app.jar}"
JAVA_OPTS="${JAVA_OPTS:--Xmx512m -Xms256m}"
SPRING_PROFILES_ACTIVE="${SPRING_PROFILES_ACTIVE:-default}"

# Display configuration
echo "Java Options: ${JAVA_OPTS}"
echo "Spring Profiles: ${SPRING_PROFILES_ACTIVE}"
echo "JAR File: ${APP_JAR}"
echo "=========================================="

# Pre-startup checks
if [ ! -f "/opt/app/${APP_JAR}" ]; then
    echo "ERROR: JAR file not found: /opt/app/${APP_JAR}"
    exit 1
fi

# Start the application
exec java ${JAVA_OPTS} \
    -Dspring.profiles.active=${SPRING_PROFILES_ACTIVE} \
    -jar /opt/app/${APP_JAR} \
    "$@"
```

### Features
- ✅ **Configurable JVM options**: Set via `JAVA_OPTS` environment variable
- ✅ **Spring profiles**: Set via `SPRING_PROFILES_ACTIVE`
- ✅ **Pre-flight checks**: Verifies JAR exists before starting
- ✅ **Signal handling**: Uses `exec` for proper signal forwarding
- ✅ **Logging**: Displays startup configuration
- ✅ **Flexible**: Accepts additional arguments via `"$@"`

### Usage Examples

#### Default startup:
```bash
docker run chat-app:1.0.0-20260129
```

#### With custom JVM options:
```bash
docker run -e JAVA_OPTS="-Xmx1g -Xms512m" chat-app:1.0.0-20260129
```

#### With Spring profile:
```bash
docker run -e SPRING_PROFILES_ACTIVE=production chat-app:1.0.0-20260129
```

#### With debugging:
```bash
docker run -e JAVA_OPTS="-Xmx512m -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005" \
    -p 5005:5005 \
    chat-app:1.0.0-20260129
```

---

## 🔐 Jenkins Credentials

### Required Credentials

#### 1. Artifactory Credentials
**Type**: Username with password
**ID**: `artifactory-credentials`
**Scope**: Global
**Usage**: 
- Maven artifact upload/download
- Docker registry authentication
- JAR download during Docker build

**Setup in Jenkins**:
```
Jenkins → Manage Jenkins → Credentials → System → Global credentials → Add Credentials

Kind: Username with password
Scope: Global
ID: artifactory-credentials
Username: <your-artifactory-username>
Password: <your-artifactory-password-or-api-token>
Description: Artifactory credentials for Maven and Docker
```

### Credential Usage

#### In Pipeline 1 (Maven):
```groovy
environment {
    ARTIFACTORY_CREDS = credentials('artifactory-credentials')
}

// Access as:
${ARTIFACTORY_CREDS_USR}  // Username
${ARTIFACTORY_CREDS_PSW}  // Password
```

#### In Pipeline 2 (Docker):
```groovy
environment {
    ARTIFACTORY_CREDS = credentials('artifactory-credentials')
}

// Passed to Docker build as build args
--build-arg ARTIFACTORY_USER=${ARTIFACTORY_CREDS_USR}
--build-arg ARTIFACTORY_PASSWORD=${ARTIFACTORY_CREDS_PSW}
```

---

## 🔄 Execution Flow

### Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ Developer commits code to Git                                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ PIPELINE 1: Maven Build (Jenkinsfile-maven)                 │
├─────────────────────────────────────────────────────────────┤
│ 1. Setup & Version                                           │
│    - Generate BUILD_LABEL: 20260129                         │
│    - Set FULL_VERSION: 1.0.0-20260129                       │
│    - Update pom.xml version                                  │
├─────────────────────────────────────────────────────────────┤
│ 2. Clean                                                     │
│    - ./mvnw clean                                            │
├─────────────────────────────────────────────────────────────┤
│ 3. Compile                                                   │
│    - ./mvnw compile                                          │
├─────────────────────────────────────────────────────────────┤
│ 4. Test                                                      │
│    - ./mvnw test                                             │
│    - Publish test results                                    │
├─────────────────────────────────────────────────────────────┤
│ 5. Package                                                   │
│    - ./mvnw package -DskipTests                             │
│    - Create: app-1.0.0-20260129.jar                         │
├─────────────────────────────────────────────────────────────┤
│ 6. Create Docker Context                                     │
│    - Package Dockerfile, entrypoint.sh into ZIP              │
│    - Create: docker-context-1.0.0-20260129.zip              │
│    - Contents: Dockerfile, .dockerignore, entrypoint.sh      │
├─────────────────────────────────────────────────────────────┤
│ 7. Deploy to Artifactory                                     │
│    - Upload app-1.0.0-20260129.jar                          │
│    - Upload docker-context-1.0.0-20260129.zip               │
│    - Location: Maven Artifactory                             │
├─────────────────────────────────────────────────────────────┤
│ 8. Trigger Docker Build                                      │
│    - Start Pipeline 2 with parameters                        │
│    - Pass: FULL_VERSION, BUILD_LABEL, etc.                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ PIPELINE 2: Docker Build (Jenkinsfile-docker)               │
├─────────────────────────────────────────────────────────────┤
│ 1. Validate Parameters                                       │
│    - Check FULL_VERSION and BUILD_LABEL provided            │
├─────────────────────────────────────────────────────────────┤
│ 2. Download Docker Context                                   │
│    - Download docker-context-1.0.0-20260129.zip             │
│    - From: Maven Artifactory                                 │
├─────────────────────────────────────────────────────────────┤
│ 3. Extract Context                                           │
│    - Unzip docker-context-1.0.0-20260129.zip                │
│    - Verify Dockerfile and entrypoint.sh exist               │
├─────────────────────────────────────────────────────────────┤
│ 4. Build Docker Image                                        │
│    ┌───────────────────────────────────────────────────┐   │
│    │ Multi-Stage Docker Build                           │   │
│    ├───────────────────────────────────────────────────┤   │
│    │ Stage 1: Downloader                                │   │
│    │ - Install curl                                     │   │
│    │ - Download app-1.0.0-20260129.jar from Artifactory│   │
│    │ - Using credentials from build args               │   │
│    ├───────────────────────────────────────────────────┤   │
│    │ Stage 2: Runtime                                   │   │
│    │ - Install Java 17 JRE                             │   │
│    │ - Copy JAR from Stage 1                           │   │
│    │ - Copy entrypoint.sh                              │   │
│    │ - Create non-root user                            │   │
│    │ - Add labels and health check                     │   │
│    │ - Set entrypoint to entrypoint.sh                 │   │
│    └───────────────────────────────────────────────────┘   │
│    - Result: chat-app:1.0.0-20260129                        │
├─────────────────────────────────────────────────────────────┤
│ 5. Tag Image                                                 │
│    - Tag: chat-app:1.0.0-20260129                           │
│    - Tag: chat-app:latest                                    │
│    - With registry prefix                                    │
├─────────────────────────────────────────────────────────────┤
│ 6. Push to Registry                                          │
│    - Login to Artifactory Docker registry                    │
│    - Push: chat-app:1.0.0-20260129                          │
│    - Push: chat-app:latest                                   │
│    - Logout                                                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Artifacts Available                                          │
├─────────────────────────────────────────────────────────────┤
│ Maven Artifactory:                                           │
│ ✅ app-1.0.0-20260129.jar                                   │
│ ✅ docker-context-1.0.0-20260129.zip                        │
│                                                              │
│ Docker Registry:                                             │
│ ✅ chat-app:1.0.0-20260129                                  │
│ ✅ chat-app:latest                                           │
└─────────────────────────────────────────────────────────────┘
```

### Manual Execution

#### Run Pipeline 1 Only:
```
1. Navigate to Jenkins → Pipeline 1
2. Click "Build with Parameters"
3. Set BASE_VERSION (e.g., 1.0.0)
4. Set TRIGGER_DOCKER_BUILD to false
5. Click "Build"
```

#### Run Pipeline 2 Manually:
```
1. Navigate to Jenkins → Pipeline 2
2. Click "Build with Parameters"
3. Set FULL_VERSION (e.g., 1.0.0-20260129)
4. Set BUILD_LABEL (e.g., 20260129)
5. Click "Build"
```

#### Run Both Pipelines (Auto):
```
1. Navigate to Jenkins → Pipeline 1
2. Click "Build with Parameters"
3. Set BASE_VERSION (e.g., 1.0.0)
4. Set TRIGGER_DOCKER_BUILD to true
5. Click "Build"
6. Pipeline 2 will start automatically
```

---

## ✅ Implementation Checklist

### Phase 1: Project Setup
- [ ] Create project directory structure
- [ ] Create `.dockerignore` file
- [ ] Create `entrypoint.sh` script
- [ ] Set up Jenkins credentials in Jenkins UI
- [ ] Verify Artifactory access (Maven and Docker)

### Phase 2: Dockerfile Creation
- [ ] Create `Dockerfile` with UBI8 multi-stage build
- [ ] Add Stage 1: Downloader with JAR download logic
- [ ] Add Stage 2: Runtime with Java 17 and application setup
- [ ] Add entrypoint.sh copy and permissions
- [ ] Add labels for traceability
- [ ] Add health check
- [ ] Test Dockerfile locally (optional)

### Phase 3: Pipeline 1 - Maven Build
- [ ] Create `Jenkinsfile-maven`
- [ ] Configure pipeline parameters
- [ ] Implement Stage 1: Setup & Version
- [ ] Implement Stage 2: Clean
- [ ] Implement Stage 3: Compile
- [ ] Implement Stage 4: Test
- [ ] Implement Stage 5: Package
- [ ] Implement Stage 6: Create Docker Context (include entrypoint.sh)
- [ ] Implement Stage 7: Deploy to Artifactory
- [ ] Implement Stage 8: Trigger Docker Build
- [ ] Add post-build actions (success/failure notifications)
- [ ] Test pipeline execution
- [ ] Verify artifacts in Artifactory

### Phase 4: Pipeline 2 - Docker Build
- [ ] Create `Jenkinsfile-docker`
- [ ] Configure pipeline parameters
- [ ] Implement Stage 1: Validate Parameters
- [ ] Implement Stage 2: Download Docker Context
- [ ] Implement Stage 3: Extract Context (verify entrypoint.sh)
- [ ] Implement Stage 4: Build Docker Image
- [ ] Implement Stage 5: Tag Image
- [ ] Implement Stage 6: Push to Registry
- [ ] Add post-build actions (cleanup, notifications)
- [ ] Test pipeline execution
- [ ] Verify Docker image in registry

### Phase 5: Integration & Testing
- [ ] Configure auto-trigger between pipelines
- [ ] Test end-to-end flow (Pipeline 1 → Pipeline 2)
- [ ] Verify version labeling is correct
- [ ] Test manual pipeline execution
- [ ] Test with different BASE_VERSION values
- [ ] Verify artifacts are properly named
- [ ] Test entrypoint.sh with different environment variables
- [ ] Test rollback scenario (rebuild old version)
- [ ] Document any issues and resolutions

### Phase 6: Documentation & Handoff
- [ ] Update README.md with usage instructions
- [ ] Document Jenkins job configuration
- [ ] Create troubleshooting guide
- [ ] Document Artifactory structure
- [ ] Create runbook for operations team
- [ ] Document entrypoint.sh environment variables

---

## 📝 Files to Create

### 1. Core Files
```
project-root/
├── Dockerfile                    # Multi-stage Docker build
├── .dockerignore                 # Files to exclude from Docker context
├── entrypoint.sh                 # Application startup script
├── Jenkinsfile-maven             # Pipeline 1: Maven build
├── Jenkinsfile-docker            # Pipeline 2: Docker build
├── plan.md                       # This file
└── README.md                     # Project documentation
```

### 2. File Descriptions

#### `Dockerfile`
- Multi-stage build with UBI8
- Stage 1: Downloads JAR from Artifactory
- Stage 2: Creates runtime image with entrypoint.sh
- ~75 lines

#### `.dockerignore`
- Excludes unnecessary files from Docker context
- Keeps context minimal
- ~30 lines

#### `entrypoint.sh`
- Application startup script
- Configurable JVM options and Spring profiles
- Pre-flight checks
- ~40 lines

#### `Jenkinsfile-maven`
- Pipeline 1: Maven build and deploy
- 8 stages
- Includes entrypoint.sh in docker-context.zip
- ~210 lines

#### `Jenkinsfile-docker`
- Pipeline 2: Docker build and push
- 6 stages
- ~180 lines

#### `plan.md`
- This comprehensive plan document
- Architecture, design, and implementation guide
- ~1100 lines

#### `README.md`
- Project overview
- Quick start guide
- Usage instructions
- ~120 lines

---

## 🎯 Success Criteria

### Pipeline 1 Success
- ✅ All Maven tests pass
- ✅ JAR file created with correct version
- ✅ docker-context.zip created (minimal: Dockerfile, .dockerignore, entrypoint.sh)
- ✅ Both artifacts uploaded to Artifactory
- ✅ Artifacts accessible via Artifactory URL
- ✅ Pipeline 2 triggered automatically (if enabled)

### Pipeline 2 Success
- ✅ docker-context.zip downloaded successfully
- ✅ Dockerfile and entrypoint.sh extracted and verified
- ✅ Docker image built without errors
- ✅ JAR downloaded from Artifactory during build
- ✅ entrypoint.sh properly copied and executable
- ✅ Image tagged with version and latest
- ✅ Images pushed to Docker registry
- ✅ Images accessible via Docker registry

### Overall Success
- ✅ End-to-end flow works (Pipeline 1 → Pipeline 2)
- ✅ Version format correct: `1.0.0-YYYYMMDD`
- ✅ All artifacts properly labeled and traceable
- ✅ No credentials exposed in Docker images
- ✅ entrypoint.sh works with environment variables
- ✅ Can rebuild any version from artifacts
- ✅ Manual and automatic execution both work

---

## 🚀 Next Steps

1. **Review this plan** - Ensure all requirements are covered
2. **Create files in order**:
   - `.dockerignore`
   - `entrypoint.sh`
   - `Dockerfile`
   - `Jenkinsfile-maven`
   - `Jenkinsfile-docker`
   - `README.md`
3. **Test each component** as it's created
4. **Integrate and test** end-to-end flow

---

## 📚 References

### Artifactory URLs
- **Maven Repository**: `https://na.artifactory.swg-devops.com/artifactory/ip-devops-team-sandbox-maven-local/`
- **Docker Registry**: `https://na.artifactory.swg-devops.com/artifactory/ip-devops-team-sandbox-dev-docker-local/`

### Base Images
- **UBI8**: `registry.access.redhat.com/ubi8/ubi:8.10`
- **Java**: Installed via `dnf install java-17-openjdk-headless`

### Jenkins Credentials
- **ID**: `artifactory-credentials`
- **Type**: Username with password
- **Scope**: Global

---

## 📞 Support

For issues or questions:
1. Check this plan document
2. Review Jenkins build logs
3. Verify Artifactory access
4. Check Docker build logs
5. Test entrypoint.sh locally
6. Consult team documentation

---

**Document Version**: 1.1
**Last Updated**: 2026-01-29
**Status**: Ready for Implementation ✅
**Changes**: Added entrypoint.sh script and updated docker-context.zip contents
