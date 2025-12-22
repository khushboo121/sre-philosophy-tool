# Security Scanning Tools and Workflows

This document provides a comprehensive overview of the security scanning tools and workflows used in the Tanzu Platform ecosystem. It differentiates between various security tools, their purposes, and how they integrate into the CI/CD pipeline.

## Table of Contents

- [Overview](#overview)
- [Security Tools Overview](#security-tools-overview)
- [Tool Comparison Matrix](#tool-comparison-matrix)
- [Coverity - Static Application Security Testing (SAST)](#coverity---static-application-security-testing-sast)
- [Black Duck - Software Composition Analysis (SCA)](#black-duck---software-composition-analysis-sca)
- [TVS - Tanzu Vulnerability Service](#tvs---tanzu-vulnerability-service)
- [Integration with CI/CD](#integration-with-cicd)
- [Best Practices](#best-practices)
- [References](#references)

## Overview

The security-scans repository contains GitHub workflows and automation scripts for performing comprehensive security scanning across Tanzu Platform services. The repository implements a multi-layered security approach using different tools to address various aspects of application security:

1. **Static Analysis** - Analyzing source code for vulnerabilities
2. **Dependency Scanning** - Identifying vulnerable open-source components
3. **Vulnerability Management** - Tracking and managing security findings
4. **License Compliance** - Ensuring open-source license compliance

## Security Tools Overview

### Tool Categories

| Category | Tool | Purpose | Scan Type |
|----------|------|---------|-----------|
| **SAST** | Coverity | Static code analysis for security vulnerabilities and code quality | Source Code |
| **SCA** | Black Duck | Open-source dependency scanning and license compliance | Dependencies/Binaries |
| **Vulnerability Management** | TVS | Centralized vulnerability tracking and reporting | All Findings |

## Tool Comparison Matrix

| Feature | Coverity | Black Duck | TVS |
|---------|----------|------------|-----|
| **Primary Purpose** | Static code analysis | Dependency scanning | Vulnerability management |
| **What It Scans** | Source code (Java, Go, JavaScript, etc.) | Dependencies, binaries, Docker images | Aggregates findings from all tools |
| **Detection Type** | Code vulnerabilities, bugs, quality issues | Known CVEs in dependencies | Centralized tracking |
| **Language Support** | Java, Go, JavaScript, C/C++, C#, PHP, Python, Ruby | All (via dependency analysis) | Language agnostic |
| **Scan Location** | Pre-build (source code) | Post-build (dependencies/binaries) | Post-analysis |
| **Output** | Defect reports, code quality metrics | CVE reports, license compliance | Unified vulnerability dashboard |
| **Integration** | GitHub Actions workflows | Jenkins jobs, GitHub Actions | CLI tool, API |
| **Frequency** | Daily scheduled scans | On-demand, triggered scans | Continuous tracking |

## Coverity - Static Application Security Testing (SAST)

### Purpose

Coverity is a Static Application Security Testing (SAST) tool that analyzes source code to identify:
- **Security Vulnerabilities**: SQL injection, XSS, buffer overflows, etc.
- **Code Quality Issues**: Memory leaks, null pointer dereferences, resource leaks
- **Defect Detection**: Logic errors, race conditions, concurrency issues
- **Compliance**: Code standards adherence, best practices violations

### How It Works

1. **Code Instrumentation**: Coverity instruments the build process to capture compilation information
2. **Static Analysis**: Analyzes the code without executing it
3. **Defect Detection**: Identifies potential security vulnerabilities and code defects
4. **Results Upload**: Commits findings to Coverity Connect server for tracking

### Supported Languages

- **Java** (via Maven, Gradle, Ant)
- **Go** (Go modules, multi-module support)
- **JavaScript/TypeScript** (Node.js, React, Angular, Vue)
- **C/C++**
- **C#**
- **PHP/Python/Ruby** (via autocapture)

### Key Features

- **Multi-Module Support**: Scans multiple Go modules in a single repository
- **Automatic Stream Creation**: Creates Coverity streams automatically if they don't exist
- **Custom Configuration**: Supports `coverity.yaml` for repository-specific settings
- **Parallel Scanning**: Scans up to 15 services simultaneously
- **Daily Scheduled Scans**: Automated daily scans at 8 AM UTC (TanzuHub) and 10 AM UTC (TPCF)

### Workflow Architecture

```
trigger-tanzuhub-coverity-scan.yml (TanzuHub Services)
         │
         ├───> tanzu-common-coverity-workflow.yml (Reusable)
         │
trigger-tpcf-coverity-scan.yml (TPCF Services)
```

### Coverage

- **TanzuHub Services**: 36 services (19 autocapture, 1 JavaScript, 16 Golang)
- **TPCF Services**: 33 active services (24 autocapture, 9 Golang)
- **Total**: 69 services scanned daily

### Use Cases

- **Pre-commit Analysis**: Identify security issues before code is merged
- **Code Quality**: Maintain consistent code quality across services
- **Compliance**: Meet security compliance requirements
- **Defect Tracking**: Track and manage code defects over time

### Example Configuration

```yaml
# coverity.yaml in repository root
service: my-service
language: golang
goVersion: 1.24.6
golang_build_command: ./...
custom_path: ./src
default_branch: main
commit: main
```

### Limitations

- **Shell Scripts**: Not supported (use ShellCheck instead)
- **Packaging Repos**: Cannot scan repositories with no compilable code
- **Build Requirements**: Requires compilable code (not just configuration files)

## Black Duck - Software Composition Analysis (SCA)

### Purpose

Black Duck is a Software Composition Analysis (SCA) tool that:
- **Identifies Open-Source Components**: Discovers all open-source dependencies in your codebase
- **Detects Known Vulnerabilities**: Matches dependencies against CVE databases
- **License Compliance**: Identifies and tracks open-source licenses
- **Policy Enforcement**: Enforces security and license policies
- **Bill of Materials (BOM)**: Generates comprehensive software bill of materials

### How It Works

1. **Dependency Discovery**: Scans project files (package.json, pom.xml, go.mod, etc.) and binaries
2. **Component Matching**: Matches discovered components against Black Duck's knowledge base
3. **Vulnerability Detection**: Identifies known CVEs in matched components
4. **License Analysis**: Analyzes licenses of all open-source components
5. **Report Generation**: Creates detailed reports with findings and recommendations

### Scan Types

- **Signature Scan**: Scans binaries and compiled artifacts
- **Binary Scan**: Analyzes binary files for embedded components
- **Docker Scan**: Scans Docker images for vulnerabilities
- **Source Scan**: Analyzes source code for dependencies

### Key Features

- **Comprehensive Database**: Access to extensive CVE and component databases
- **License Compliance**: Tracks and reports on license obligations
- **Policy Management**: Enforces security and license policies
- **Integration**: Integrates with Jenkins, GitHub Actions, and CI/CD pipelines
- **Notice File Generation**: Automatically generates license notice files

### Scripts and Automation

#### 1. BlackDuck-Scan-Tanzu-CLI-Plugin.py
**Purpose**: Triggers Black Duck scans for Tanzu CLI plugins via Jenkins

**Functionality**:
- Reads Docker image paths from JSON configuration
- Triggers Jenkins job `tnz-blackduck-scan-job` for each plugin
- Configures scan parameters (project name, version, phase, scan type)

**Usage**:
```bash
python BlackDuck-Scan-Tanzu-CLI-Plugin.py
```

#### 2. BlackDuck_Gen_NoticeFile.py
**Purpose**: Generates license notice files for projects

**Functionality**:
- Authenticates with Black Duck API
- Retrieves project and version information
- Generates comprehensive notice files with copyright and license information
- Supports concurrent processing for multiple projects

**Usage**:
```bash
export BD_PROD_URL="https://<org>.app.blackduck.com"
export BD_API_TOKEN="your-token"
export release_head="release-head"
python BlackDuck_Gen_NoticeFile.py
```

#### 3. BlackDuck_Update_Phase.py
**Purpose**: Updates Black Duck project version phases

**Functionality**:
- Updates project version phases (DEVELOPMENT, RELEASED, etc.)
- Supports bulk updates based on project name patterns
- Sets distribution type (SAAS, EXTERNAL) based on project name

**Usage**:
```bash
export BD_PROD_URL="https://broadcom.app.blackduck.com"
export BD_API_TOKEN="your-token"
python BlackDuck_Update_Phase.py \
  --phase "RELEASED" \
  --prefix "TNZ-*" \
  --versionName "release-head"
```

#### 4. BlackDuck_Gen_Tanzu_CLI_Path.py
**Purpose**: Generates Tanzu CLI paths for Black Duck scanning

**Functionality**:
- Creates configuration files for Tanzu CLI plugin scanning
- Maps service names to Docker image paths

#### 5. notices-report-generator_concurrentProcEnabled.py
**Purpose**: Generates comprehensive notices reports with concurrent processing

**Functionality**:
- Generates detailed notices reports from Black Duck
- Supports concurrent processing for performance
- Includes copyright information and license details
- Exports to various formats

**Usage**:
```bash
python notices-report-generator_concurrentProcEnabled.py \
  --bdurl https://broadcom.app.blackduck.com \
  --apitoken "<apitoken>" \
  -p "<Projname>" \
  -v <projversion> \
  --exclude_usage '' \
  --copyright true \
  --output_file <outputfilename>
```

### Use Cases

- **Dependency Vulnerability Scanning**: Identify CVEs in open-source dependencies
- **License Compliance**: Ensure compliance with open-source license requirements
- **Software Bill of Materials**: Generate comprehensive SBOMs for compliance
- **Policy Enforcement**: Enforce security and license policies across projects
- **Docker Image Scanning**: Scan container images for vulnerabilities

### Integration Points

- **Jenkins**: Automated scans via `tnz-blackduck-scan-job`
- **GitHub Actions**: Workflow-based scanning (see `generate_trigger.py`)
- **Docker Registry**: Direct image scanning from JFrog/Artifactory
- **CI/CD Pipelines**: Integrated into build and release processes

## TVS - Tanzu Vulnerability Service

### Purpose

TVS (Tanzu Vulnerability Service) is a centralized vulnerability management platform that:
- **Aggregates Findings**: Collects vulnerability data from multiple sources
- **Tracks Builds**: Associates vulnerabilities with specific build versions
- **Manages Lifecycle**: Tracks vulnerability status through remediation lifecycle
- **Provides Reporting**: Centralized dashboard for vulnerability reporting
- **Component Management**: Manages component metadata and relationships

### How It Works

1. **Build Submission**: Submits build information via YAML files or API
2. **Vulnerability Aggregation**: Collects findings from Coverity, Black Duck, and other sources
3. **Build Tracking**: Associates vulnerabilities with specific build versions
4. **Status Management**: Tracks vulnerability status (new, acknowledged, remediated)
5. **Reporting**: Provides dashboards and reports for stakeholders

### Key Features

- **Centralized Management**: Single source of truth for all vulnerabilities
- **Build Association**: Links vulnerabilities to specific build versions
- **Lifecycle Tracking**: Tracks vulnerabilities through remediation process
- **API Access**: RESTful API for integration and automation
- **CLI Tool**: Command-line interface for common operations

### TVS Helper Script (tvs-helper.sh)

The `tvs-helper.sh` script provides utility functions for TVS operations:

#### Functions

1. **get_build_id(component_name, build_version, [isGa])**
   - Retrieves build ID for a given component and version
   - Supports GA (Generally Available) flag for production builds

2. **submit_tvs_build(yaml_file_path)**
   - Submits a TVS build from YAML configuration file
   - Automates build submission process

3. **get_build(build_id)**
   - Retrieves detailed build information by build ID
   - Returns JSON with build details and associated vulnerabilities

4. **get_component_id(component_name)**
   - Retrieves component ID given a component name
   - Useful for API operations requiring component IDs

5. **get_latest_build_id(component_name, release_line, [build_type])**
   - Gets the latest build ID for a component and release line
   - Supports filtering by build type (DEV, GA, etc.)
ƒ
#### Usage

```bash
# Source the helper script
source ./scripts/tvs-helper.sh

# Get build ID
BUILD_ID=$(get_build_id "my-component" "1.0.0")

# Submit build
submit_tvs_build "./build-config.yaml"

# Get build details
BUILD_DETAILS=$(get_build "$BUILD_ID")
```

### Environment Configuration

```bash
# Set environment (production or staging)
export TVS_ENVIRONMENT="production"  # or "staging"

# API URLs are automatically set:
# Production: https://tpe-tvs.acc.broadcom.net
# Staging: https://tpe-tvs-staging.acc.broadcom.net
```

### Use Cases

- **Vulnerability Aggregation**: Centralize findings from multiple scanning tools
- **Build Tracking**: Associate vulnerabilities with specific releases
- **Remediation Management**: Track vulnerability remediation progress
- **Compliance Reporting**: Generate reports for security compliance
- **Release Gating**: Use vulnerability data for release decisions

## Integration with CI/CD

### Workflow Integration

#### Coverity Integration

```yaml
# GitHub Actions workflow
- name: Coverity Scan
  uses: TNZ/tpe-coverity-scan/scan@v2
  with:
    action: "scan"
    service: "my-service"
    code-base-language-type: "golang"
    go-version: "1.24.6"
    coverity-stream: "RID__0000__my-service-main"
```

#### Black Duck Integration

```yaml
# Jenkins job trigger
- name: Trigger Black Duck Scan
  run: |
    python scripts/BlackDuck-Scan-Tanzu-CLI-Plugin.py
```

#### TVS Integration

```yaml
# TVS build submission
- name: Submit to TVS
  run: |
    source scripts/tvs-helper.sh
    submit_tvs_build "./tvs-build-config.yaml"
```

### Scan Frequency

| Tool | Frequency | Trigger |
|------|-----------|---------|
| **Coverity** | Daily | Scheduled (8 AM UTC TanzuHub, 10 AM UTC TPCF) |
| **Black Duck** | On-demand | Manual trigger, release builds |
| **TVS** | Continuous | After each scan completion |

### Notification and Reporting

- **Coverity**: Google Chat notifications on scan failures
- **Black Duck**: Email/Slack notifications for policy violations
- **TVS**: Dashboard-based reporting and API access

## Best Practices

### 1. Coverity Best Practices

- **Use coverity.yaml**: Keep configuration with code for version control
- **Correct Language Type**: Use `golang` for Go projects, not `autocapture`
- **Match Go Versions**: Ensure `goVersion` matches `go.mod` requirement
- **Multi-Module Support**: Use `multi_module_paths` for monorepos
- **Regular Reviews**: Review Coverity Connect for new defects regularly

### 2. Black Duck Best Practices

- **Regular Scans**: Scan dependencies regularly, especially before releases
- **Policy Enforcement**: Define and enforce security and license policies
- **Notice Files**: Generate and maintain accurate notice files
- **Version Tracking**: Keep track of dependency versions and updates
- **Remediation**: Prioritize and remediate high-severity vulnerabilities

### 3. TVS Best Practices

- **Consistent Naming**: Use consistent component and build naming conventions
- **Build Association**: Always associate vulnerabilities with specific builds
- **Status Updates**: Keep vulnerability status up-to-date
- **Regular Reporting**: Generate regular reports for stakeholders
- **Integration**: Integrate TVS into release processes

### 4. General Security Scanning Best Practices

- **Shift Left**: Scan early in the development lifecycle
- **Automate**: Automate scans in CI/CD pipelines
- **Prioritize**: Focus on high-severity vulnerabilities first
- **Remediate**: Have a clear remediation process
- **Document**: Document security findings and remediation steps
- **Review**: Regularly review and update scanning configurations

## Security Tool Selection Guide

### When to Use Coverity

- ✅ Analyzing source code for security vulnerabilities
- ✅ Detecting code quality issues and bugs
- ✅ Ensuring code standards compliance
- ✅ Pre-commit security checks
- ❌ Not for dependency scanning
- ❌ Not for binary/Docker image scanning

### When to Use Black Duck

- ✅ Scanning open-source dependencies
- ✅ Identifying known CVEs in dependencies
- ✅ License compliance checking
- ✅ Docker image vulnerability scanning
- ✅ Generating software bill of materials (SBOM)
- ❌ Not for source code analysis
- ❌ Not for custom code vulnerabilities

### When to Use TVS

- ✅ Centralized vulnerability management
- ✅ Aggregating findings from multiple tools
- ✅ Build-level vulnerability tracking
- ✅ Compliance reporting
- ✅ Release gating decisions
- ❌ Not a scanning tool (aggregation only)

## Tool Integration Workflow

```
┌─────────────────┐
│  Source Code    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Coverity      │───► SAST Findings
│   (SAST Scan)   │
└─────────────────┘
         │
         ▼
┌─────────────────┐
│  Build Process  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Black Duck     │───► SCA Findings
│  (SCA Scan)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│      TVS        │───► Unified Dashboard
│ (Aggregation)   │
└─────────────────┘
```

## References

### Documentation

- [Coverity Scan Workflows Documentation](./coverity-readme.md) - Comprehensive Coverity documentation
- [Black Duck Documentation](https://www.synopsys.com/software-integrity/software-security-tools/black-duck-sourcery.html) - Official Black Duck documentation
- [TVS CLI Documentation](https://tpe-tvs.acc.broadcom.net/docs) - TVS API and CLI documentation

### Repository Structure

```
security-scans/
├── README.md                          # Main repository README
├── coverity-readme.md                 # Coverity detailed documentation
├── scripts/
│   ├── BlackDuck-Scan-Tanzu-CLI-Plugin.py
│   ├── BlackDuck_Gen_NoticeFile.py
│   ├── BlackDuck_Gen_Tanzu_CLI_Path.py
│   ├── BlackDuck_Update_Phase.py
│   ├── generate_trigger.py
│   ├── notices-report-generator_concurrentProcEnabled.py
│   ├── tvs-helper.sh
│   └── others/
│       ├── generate_trigger.py
│       └── pull_image_from_jfrog.py
└── .github/
    └── workflows/
        ├── trigger-tanzuhub-coverity-scan.yml
        ├── trigger-tpcf-coverity-scan.yml
        ├── tanzu-common-coverity-workflow.yml
        └── [Black Duck workflows]
```

### Key URLs

- **Coverity Connect**: https://cov-vmw-tnz.devops.broadcom.net:8443/
- **Black Duck**: https://broadcom.app.blackduck.com
- **TVS Production**: https://tpe-tvs.acc.broadcom.net
- **TVS Staging**: https://tpe-tvs-staging.acc.broadcom.net
- **GitHub Repository**: https://github.gwd.broadcom.net/TNZ/security-scans

## Summary

The security-scans repository implements a comprehensive security scanning strategy using three complementary tools:

1. **Coverity (SAST)**: Analyzes source code for security vulnerabilities and code quality issues
2. **Black Duck (SCA)**: Scans dependencies and binaries for known CVEs and license compliance
3. **TVS (Vulnerability Management)**: Aggregates and manages findings from all security tools

Together, these tools provide:
- **Comprehensive Coverage**: Source code, dependencies, and binaries
- **Automated Scanning**: Daily scheduled scans and on-demand triggers
- **Centralized Management**: Unified vulnerability tracking and reporting
- **Compliance Support**: License compliance and security policy enforcement

By using these tools in combination, organizations can achieve a robust security posture that covers all aspects of application security from source code to deployed artifacts.

---

**Last Updated**: December 2024  
**Maintainer**: TNZ DevOps Team  
**Repository**: [security-scans](https://github.gwd.broadcom.net/TNZ/security-scans)

