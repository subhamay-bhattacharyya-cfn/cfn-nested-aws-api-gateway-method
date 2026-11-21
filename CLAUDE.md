# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **CloudFormation template repository** that provides a reusable nested stack template for creating an API Gateway REST API method backed by a Lambda function. The template is designed to be referenced by parent CloudFormation stacks.

**Key characteristics:**

- Nested CloudFormation template (referenced via `TemplateURL`)
- Optional REST API and path resource creation (CI builds) or attachment to an existing API (non-CI builds)
- Lambda proxy integration (`AWS_PROXY`) or Lambda `AWS` integration
- Authorization defaults to `AWS_IAM` (SigV4), for AgentCore Gateway and direct callers
- Automated semantic versioning and releases
- AWS OIDC authentication for CI/CD deployments

## Project Structure

```text
cloudformation/
├── template.yaml                 # Nested template: REST API, path resource, method
├── parameters.json               # Example parameter values
└── stack-config.json             # Stack name, template, and parameter file references

.github/workflows/
├── ci.yaml                       # CI workflow
├── release.yaml                  # Semantic release on push to main
└── create-branch.yaml            # Auto-create feature branches from issues

scripts/plugins/
├── release.config.js             # Semantic-release configuration
└── (other release plugins)       # Custom commit analysis, notes generation

.claude/
├── .skills/                      # Project skills
└── settings.json                 # Claude Code workspace settings

.devcontainer/
└── devcontainer.json             # Dev container setup (Node.js 20)

package.json                      # Dependencies: semantic-release, commitizen
README.md                         # Template documentation and usage examples
```

## Development Commands

### Install dependencies

```bash
npm ci
```

### Trigger semantic release (usually automatic on main)

```bash
npm run release
```

### Commit with conventional commit format

```bash
npx cz commit
```

Select `feat`, `fix`, or `chore` type. Only `feat` and `fix` trigger releases.

## Key Architecture Concepts

### Nested Stack Pattern

This repo provides a **nested stack template** that is referenced from parent/root CloudFormation stacks via `TemplateURL`. The template is self-contained and exports its method ID as an output.

- **Parent stack** calls: `AWS::CloudFormation::Stack` with `TemplateURL` pointing to S3
- **Nested template** returns values via `Outputs` (currently `RestApiMethodId`)
- Parent retrieves outputs via `!GetAtt NestedStack.Outputs.OutputKey`

### CI Build vs Existing API

The `CiSuffix` parameter selects the mode:

- **CI build** (`CiSuffix` non-empty, `IsCiBuild` condition true): creates `RestApi` (REGIONAL, tagged with `ProjectName` and `Environment`) and `RestApiResource` (path segment set by `PathPart`). The method attaches to `RestApiResource`.
- **Non-CI build** (`CiSuffix` empty): attaches the method to an existing API (`RestApiId`) and resource (`ExistingResourceId`). `ParentId` is also required. The `Rules` section enforces these.

### Method Integration

`RestApiMethod` is always created. Its settings are parameterised:

- `HttpMethod`, `AuthorizationType`
- `IntegrationType` (`AWS_PROXY` or `AWS`) and `IntegrationHttpMethod` (`POST` for Lambda)
- `LambdaFunctionArn` (required, validated against a Lambda function ARN pattern)

The integration `Uri` is built from `LambdaFunctionArn`. The Lambda function must grant `apigateway.amazonaws.com` invoke permission. The template does not create that permission.

## Key Files to Understand

### `cloudformation/template.yaml`

**Purpose:** Creates an API Gateway REST API method with an optional REST API and path resource

**Key inputs:**

- `ProjectName`, `Environment`: naming and tagging
- `CiSuffix`: selects CI build vs existing API mode
- `PathPart`: path segment created in CI builds (default `orders`)
- `RestApiId`, `ParentId`, `ExistingResourceId`: existing API inputs for non-CI builds
- `HttpMethod`, `AuthorizationType`, `LambdaFunctionArn`, `IntegrationType`, `IntegrationHttpMethod`: method and integration settings

**Key outputs:**

- `RestApiMethodId`: ID of the method

**Features:**

- REGIONAL REST API when in CI mode
- Conditional `RestApiId` and `ResourceId` on the method
- Rules that require existing API inputs when `CiSuffix` is empty

### `cloudformation/parameters.json` and `cloudformation/stack-config.json`

- `parameters.json` holds example parameter values for deployment
- `stack-config.json` names the stack and points to the template and parameter file

## Testing & Validation

**Manual template validation:**

```bash
aws cloudformation validate-template --template-body file://cloudformation/template.yaml
```

**Manual stack deployment (CI build):**

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name my-api-method-ci \
  --parameter-overrides \
    ProjectName=myproject \
    Environment=devl \
    CiSuffix=pipeline-12345 \
    LambdaFunctionArn=arn:aws:lambda:us-east-1:123456789012:function:list-orders \
  --region us-east-1
```

## AWS Credentials & Environment Variables

**GitHub environment variables required in `ci` environment:**

- `AWS_REGION`: CloudFormation deployment region
- `AWS_ACCOUNT_ID`: AWS account to deploy into
- `OIDC_ROLE_NAME`: IAM role name for OIDC trust (uses `arn:aws:iam::{ACCOUNT_ID}:role/{ROLE_NAME}`)
- `CFN_TEMPLATES_S3_BUCKET`: S3 bucket where templates are stored

**OIDC setup:** The CI workflow uses AWS OIDC for keyless auth. The GitHub OIDC provider must trust the specified role.

## Conventional Commits & Release Flow

This repo enforces conventional commits to drive semantic versioning:

```bash
npx cz commit
```

Commit types:

- `feat: add support for X` → triggers MINOR release
- `fix: correct behavior of Y` → triggers PATCH release
- `chore: update deps` → no release
- `docs: clarify README` → no release

Only commits to `main` trigger releases. Feature branches use this format but releases happen on merge to main.

## When Modifying Templates

1. **Edit `cloudformation/template.yaml`**
2. **Update `cloudformation/parameters.json`** if new parameters are added or required parameters change
3. **Test locally** with `aws cloudformation validate-template` (and `cfn-lint` if available)
4. **Create a PR** with conventional commit message (e.g., `feat: add authorizer parameter`)
5. **Merge to main** → release workflow creates version tag and GitHub release

## Dev Container

Pre-configured with:

- Node.js 20
- GitHub Copilot extension

Use via VS Code: `code --remote-container-url <repo-url>`

## Current Branch

Main branch is the release branch. Feature work branches from here and merges back via PR. Branch naming follows: `{type}/CFN-{issue-number}-{slug}` (e.g., `feature/CFN-42-add-encryption`).
