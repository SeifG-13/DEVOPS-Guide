# Build Automation

## What is Build Automation?

Build automation is the process of scripting and automating the compilation, testing, and packaging of software.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Build Automation Flow                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Source Code                                                   │
│       │                                                         │
│       ▼                                                         │
│   ┌─────────────┐                                              │
│   │  Checkout   │  Clone/fetch source code                     │
│   └──────┬──────┘                                              │
│          ▼                                                      │
│   ┌─────────────┐                                              │
│   │ Dependencies│  Install libraries/packages                  │
│   └──────┬──────┘                                              │
│          ▼                                                      │
│   ┌─────────────┐                                              │
│   │   Compile   │  Build source code                           │
│   └──────┬──────┘                                              │
│          ▼                                                      │
│   ┌─────────────┐                                              │
│   │    Test     │  Run automated tests                         │
│   └──────┬──────┘                                              │
│          ▼                                                      │
│   ┌─────────────┐                                              │
│   │   Package   │  Create deployable artifacts                 │
│   └──────┬──────┘                                              │
│          ▼                                                      │
│   Build Artifacts                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Build Tools Overview

### Language-Specific Build Tools

| Language | Build Tools |
|----------|-------------|
| Java | Maven, Gradle, Ant |
| JavaScript | npm, Yarn, pnpm |
| Python | pip, Poetry, setuptools |
| .NET | MSBuild, dotnet CLI |
| Go | go build |
| Rust | Cargo |
| C/C++ | Make, CMake |

## Maven (Java)

### Project Structure

```
my-project/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/
│   │   └── resources/
│   └── test/
│       ├── java/
│       └── resources/
└── target/
```

### pom.xml Example

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.10.0</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.1.2</version>
            </plugin>
        </plugins>
    </build>
</project>
```

### Maven Commands

```bash
# Clean
mvn clean

# Compile
mvn compile

# Run tests
mvn test

# Package (create JAR/WAR)
mvn package

# Install to local repo
mvn install

# Deploy to remote repo
mvn deploy

# Skip tests
mvn package -DskipTests

# Run with profile
mvn package -P production

# Generate dependency tree
mvn dependency:tree
```

### Maven in CI Pipeline

```yaml
# Jenkins
pipeline {
    agent any
    tools {
        maven 'Maven 3.9'
        jdk 'JDK 17'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
    post {
        always {
            junit 'target/surefire-reports/*.xml'
            archiveArtifacts 'target/*.jar'
        }
    }
}
```

## Gradle (Java)

### Project Structure

```
my-project/
├── build.gradle
├── settings.gradle
├── gradle/
│   └── wrapper/
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
└── src/
    ├── main/
    └── test/
```

### build.gradle Example

```groovy
plugins {
    id 'java'
    id 'application'
}

group = 'com.example'
version = '1.0.0'

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'com.google.guava:guava:32.1.2-jre'
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.0'
}

application {
    mainClass = 'com.example.Main'
}

test {
    useJUnitPlatform()
}

jar {
    manifest {
        attributes 'Main-Class': 'com.example.Main'
    }
}
```

### Gradle Commands

```bash
# Build
./gradlew build

# Clean
./gradlew clean

# Test
./gradlew test

# Run application
./gradlew run

# Create JAR
./gradlew jar

# List tasks
./gradlew tasks

# Build without tests
./gradlew build -x test

# Parallel build
./gradlew build --parallel
```

## npm/Node.js

### package.json Example

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "description": "My application",
  "main": "dist/index.js",
  "scripts": {
    "start": "node dist/index.js",
    "dev": "nodemon src/index.ts",
    "build": "tsc",
    "test": "jest",
    "test:coverage": "jest --coverage",
    "lint": "eslint src/",
    "lint:fix": "eslint src/ --fix",
    "format": "prettier --write src/",
    "clean": "rm -rf dist",
    "prebuild": "npm run clean",
    "postbuild": "npm run test"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "@types/node": "^20.8.0",
    "typescript": "^5.2.2",
    "jest": "^29.7.0",
    "eslint": "^8.50.0",
    "prettier": "^3.0.3"
  }
}
```

### npm Commands

```bash
# Install dependencies
npm install
npm ci  # Clean install (for CI)

# Run scripts
npm run build
npm run test
npm start

# Update dependencies
npm update
npm outdated

# Security audit
npm audit
npm audit fix

# Publish package
npm publish

# Version management
npm version patch  # 1.0.0 -> 1.0.1
npm version minor  # 1.0.0 -> 1.1.0
npm version major  # 1.0.0 -> 2.0.0
```

### npm in CI Pipeline

```yaml
# GitHub Actions
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm test
      - run: npm run build
```

## Python Build Tools

### requirements.txt

```text
Flask==3.0.0
requests>=2.31.0,<3.0.0
pytest==7.4.2
black==23.9.1
```

### pyproject.toml (Modern)

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "my-package"
version = "1.0.0"
description = "My Python package"
requires-python = ">=3.9"
dependencies = [
    "flask>=3.0.0",
    "requests>=2.31.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "black>=23.9.0",
    "mypy>=1.5.0",
]

[project.scripts]
my-app = "my_package.cli:main"

[tool.black]
line-length = 88

[tool.pytest.ini_options]
testpaths = ["tests"]
```

### Python Build Commands

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Linux/Mac
venv\Scripts\activate     # Windows

# Install dependencies
pip install -r requirements.txt
pip install -e ".[dev]"  # Install with dev dependencies

# Run tests
pytest
pytest --cov=src

# Linting
black src/
flake8 src/
mypy src/

# Build package
python -m build

# Upload to PyPI
twine upload dist/*
```

### Python in CI Pipeline

```yaml
# GitHub Actions
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'
      - run: pip install -e ".[dev]"
      - run: black --check src/
      - run: pytest --cov=src
```

## .NET Build Tools

### Project File (.csproj)

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\MyLibrary\MyLibrary.csproj" />
  </ItemGroup>

</Project>
```

### dotnet CLI Commands

```bash
# Create new project
dotnet new console -n MyApp
dotnet new webapi -n MyApi
dotnet new classlib -n MyLib

# Restore dependencies
dotnet restore

# Build
dotnet build
dotnet build --configuration Release

# Run
dotnet run

# Test
dotnet test
dotnet test --collect:"XPlat Code Coverage"

# Publish
dotnet publish -c Release -o ./publish

# Pack NuGet
dotnet pack -c Release

# Clean
dotnet clean
```

### .NET in CI Pipeline

```yaml
# Azure Pipelines
steps:
  - task: DotNetCoreCLI@2
    displayName: 'Restore'
    inputs:
      command: 'restore'

  - task: DotNetCoreCLI@2
    displayName: 'Build'
    inputs:
      command: 'build'
      arguments: '--configuration Release'

  - task: DotNetCoreCLI@2
    displayName: 'Test'
    inputs:
      command: 'test'
      arguments: '--configuration Release --collect:"XPlat Code Coverage"'

  - task: DotNetCoreCLI@2
    displayName: 'Publish'
    inputs:
      command: 'publish'
      arguments: '--configuration Release --output $(Build.ArtifactStagingDirectory)'
```

## Make (C/C++/General)

### Makefile Example

```makefile
# Variables
CC = gcc
CFLAGS = -Wall -Wextra -O2
SRC_DIR = src
BUILD_DIR = build
TARGET = myapp

# Source files
SOURCES = $(wildcard $(SRC_DIR)/*.c)
OBJECTS = $(SOURCES:$(SRC_DIR)/%.c=$(BUILD_DIR)/%.o)

# Default target
all: $(TARGET)

# Build target
$(TARGET): $(OBJECTS)
	$(CC) $(OBJECTS) -o $@

# Compile source files
$(BUILD_DIR)/%.o: $(SRC_DIR)/%.c | $(BUILD_DIR)
	$(CC) $(CFLAGS) -c $< -o $@

# Create build directory
$(BUILD_DIR):
	mkdir -p $(BUILD_DIR)

# Clean
clean:
	rm -rf $(BUILD_DIR) $(TARGET)

# Run tests
test: $(TARGET)
	./run_tests.sh

# Install
install: $(TARGET)
	cp $(TARGET) /usr/local/bin/

# Phony targets
.PHONY: all clean test install
```

### Make Commands

```bash
# Build default target
make

# Build specific target
make test

# Clean
make clean

# Parallel build
make -j4

# Dry run (show commands)
make -n

# Set variable
make CC=clang
```

## Build Optimization

### Caching Dependencies

```yaml
# GitHub Actions - npm
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-

# Maven
- uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}

# Gradle
- uses: actions/cache@v4
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*') }}
```

### Parallel Builds

```yaml
# Matrix builds in GitHub Actions
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node: [18, 20]
  fail-fast: false

# Maven parallel
mvn -T 4 clean package  # 4 threads
mvn -T 1C clean package # 1 thread per CPU core

# Gradle parallel
./gradlew build --parallel
```

### Incremental Builds

```yaml
# Only build changed files
# Maven (automatic with proper configuration)
mvn compile

# Gradle (automatic)
./gradlew build

# Make (automatic based on timestamps)
make
```

## Build Artifacts

### Creating Artifacts

```yaml
# Jenkins
archiveArtifacts artifacts: 'target/*.jar', fingerprint: true

# GitHub Actions
- uses: actions/upload-artifact@v4
  with:
    name: build-artifacts
    path: |
      dist/
      !dist/**/*.map
    retention-days: 5

# Azure Pipelines
- publish: $(Build.ArtifactStagingDirectory)
  artifact: drop
```

### Artifact Naming

```bash
# Include version and build info
myapp-1.0.0-build123.jar
myapp-1.0.0-20240115-abc1234.tar.gz

# Example in build script
VERSION=$(cat VERSION)
BUILD_NUM=${BUILD_NUMBER:-local}
GIT_SHA=$(git rev-parse --short HEAD)
ARTIFACT_NAME="myapp-${VERSION}-${BUILD_NUM}-${GIT_SHA}.tar.gz"
```

## Quick Reference

### Build Commands

| Tool | Build | Test | Package |
|------|-------|------|---------|
| Maven | `mvn compile` | `mvn test` | `mvn package` |
| Gradle | `./gradlew build` | `./gradlew test` | `./gradlew jar` |
| npm | `npm run build` | `npm test` | `npm pack` |
| pip | - | `pytest` | `python -m build` |
| dotnet | `dotnet build` | `dotnet test` | `dotnet publish` |

### CI/CD Best Practices

| Practice | Description |
|----------|-------------|
| Cache dependencies | Speed up builds |
| Parallel builds | Reduce build time |
| Incremental builds | Only rebuild changed |
| Artifact versioning | Track build outputs |
| Build reproducibility | Same input = same output |

---

**Previous:** [14-azure-devops.md](14-azure-devops.md) | **Next:** [16-testing-in-ci.md](16-testing-in-ci.md)
