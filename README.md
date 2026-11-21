# CloudFormation Nested Stack Template : API Gateway Rest API Method

<!-- Row 1: Status - Most Important -->
[![Release](https://img.shields.io/github/v/release/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-method?label=Release)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-method/releases)&nbsp;[![Release Workflow](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-method/actions/workflows/release.yaml/badge.svg)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-method/actions/workflows/release.yaml)&nbsp;[![Issues](https://img.shields.io/github/issues/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-method)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-method/issues)&nbsp;[![Last Commit](https://img.shields.io/github/last-commit/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-method)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-method/commits)

<!-- Row 2: Code Quality -->
[![Top Language](https://img.shields.io/badge/Languages-YAML-blue)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-method)&nbsp;[![Commits](https://img.shields.io/github/commit-activity/t/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-method)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-method/commits)

<!-- Row 3: Tech Stack -->
[![CloudFormation](https://img.shields.io/badge/CloudFormation-IaC-orange?logo=amazonaws&logoColor=white)](https://aws.amazon.com/cloudformation/)&nbsp;[![Built with Claude Code](https://img.shields.io/badge/Built_with-Claude_Code-D97757?logo=anthropic&logoColor=white)](https://claude.ai/)

<!-- Row 4: Repository Info -->
[![Files](https://img.shields.io/github/directory-file-count/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-method)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-method)&nbsp;[![Repo Size](https://img.shields.io/github/repo-size/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-method)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-method)&nbsp;[![Release Date](https://img.shields.io/github/release-date/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-method)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-method/releases)

<!-- Row 5: Custom Metrics -->
[![Custom Endpoint](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/bsubhamay/b8c944569c897ebfa5065dad9bcd3fd0/raw/cfn-nested-aws-api-gateway-method.json)](https://gist.github.com/subhamay-bhattacharyya-cfn/b8c944569c897ebfa5065dad9bcd3fd0)

---

This repository contains a nested CloudFormation template that creates an API Gateway REST API method (with an optional REST API and path resource) backed by a Lambda function through an `AWS_PROXY` integration.

## Overview

This is a **nested stack template** designed to be invoked from a parent/root CloudFormation stack. The template lives at `cloudformation/template.yaml` and should be uploaded to S3 (or passed inline) for reference by parent stacks.

## Template Files

- **`cloudformation/template.yaml`** — Nested template for the REST API, path resource, and method
- **`cloudformation/parameters.json`** — Example parameter values
- **`cloudformation/stack-config.json`** — Stack name, template, and parameter file references

## Template Features

- ✅ Creates a REGIONAL REST API, path resource, and method when `CiSuffix` is set (CI build)
- ✅ Attaches the method to an existing REST API and resource when `CiSuffix` is empty
- ✅ Lambda proxy integration (`AWS_PROXY`) or Lambda `AWS` integration
- ✅ Authorization defaults to `AWS_IAM` (SigV4)
- ✅ Parameterised HTTP method, authorization, integration type, and integration HTTP method
- ✅ Rules that require existing API inputs when not a CI build

## Parameters

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `ProjectName` | String | `cfn-nested` | Name prefix (lowercase letters, numbers, hyphens; max 20 chars) |
| `Environment` | String | `devl` | Deployment environment (`devl`, `test`, `prod`) |
| `CiSuffix` | String | `""` | Non-empty marks a CI build, which creates the REST API. Leave empty to use existing API inputs |
| `PathPart` | String | `orders` | Path segment created under the REST API root (CI builds only) |
| `RestApiId` | String | `""` | Existing REST API ID. Required when `CiSuffix` is empty |
| `ParentId` | String | `""` | Parent resource ID for `PathPart`. Required when `CiSuffix` is empty |
| `ExistingResourceId` | String | `""` | Existing resource ID the method attaches to. Required when `CiSuffix` is empty |
| `HttpMethod` | String | `GET` | Method: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`, `ANY` |
| `AuthorizationType` | String | `AWS_IAM` | `AWS_IAM` or `NONE` |
| `LambdaFunctionArn` | String | — | ARN of the Lambda function to invoke (required) |
| `IntegrationType` | String | `AWS_PROXY` | `AWS_PROXY` or `AWS` |
| `IntegrationHttpMethod` | String | `POST` | HTTP method API Gateway uses to call Lambda (`POST` required for Lambda) |

## Outputs

- `RestApiMethodId` — ID of the API Gateway method

## Usage

### 1. Reference from Parent Stack

In your parent/root CloudFormation template:

```yaml
OrdersMethodNestedStack:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: https://s3.amazonaws.com/your-cfn-bucket/templates/template.yaml
    Parameters:
      ProjectName: !Ref ProjectName
      Environment: !Ref Environment
      CiSuffix: !Ref CiSuffix
      PathPart: orders
      RestApiId: !Ref RestApiId
      ParentId: !Ref ParentId
      ExistingResourceId: !Ref ExistingResourceId
      HttpMethod: GET
      AuthorizationType: AWS_IAM
      LambdaFunctionArn: !GetAtt ListOrdersFunction.Arn
      IntegrationType: AWS_PROXY
      IntegrationHttpMethod: POST
    Tags:
      - Key: Environment
        Value: !Ref Environment

Outputs:
  MethodId:
    Value: !GetAtt OrdersMethodNestedStack.Outputs.RestApiMethodId
```

### 2. Deploy Using AWS CLI

```bash
# CI build: creates the REST API, path resource, and method
aws cloudformation create-stack \
  --stack-name cfn-nested-aws-api-gateway-method-ci \
  --template-body file://cloudformation/template.yaml \
  --parameters \
    ParameterKey=ProjectName,ParameterValue=myproject \
    ParameterKey=Environment,ParameterValue=devl \
    ParameterKey=CiSuffix,ParameterValue=pipeline-12345 \
    ParameterKey=LambdaFunctionArn,ParameterValue=arn:aws:lambda:us-east-1:123456789012:function:list-orders

# Non-CI build: attaches the method to an existing REST API and resource
aws cloudformation create-stack \
  --stack-name cfn-nested-aws-api-gateway-method-devl \
  --template-body file://cloudformation/template.yaml \
  --parameters \
    ParameterKey=ProjectName,ParameterValue=myproject \
    ParameterKey=Environment,ParameterValue=devl \
    ParameterKey=RestApiId,ParameterValue=abc123xyz \
    ParameterKey=ParentId,ParameterValue=root456 \
    ParameterKey=ExistingResourceId,ParameterValue=res789 \
    ParameterKey=LambdaFunctionArn,ParameterValue=arn:aws:lambda:us-east-1:123456789012:function:list-orders
```

The Lambda function must also grant `apigateway.amazonaws.com` permission to invoke it. This template does not create that permission.

## Resource Naming

With a non-empty `CiSuffix`, the REST API is named:

```bash
{ProjectName}-api-{AccountId}-{Environment}-{Region}-{CiSuffix}
```

Example: `myproject-api-123456789012-devl-us-east-1-pipeline-12345`

## Best Practices Implemented

- ✅ Authorization defaults to AWS IAM (SigV4)
- ✅ Parameter validation for the Lambda ARN, HTTP method, and integration settings
- ✅ REST API tagged with `ProjectName` and `Environment` for CI builds
- ✅ Conditional logic so the same template works for CI and existing-API deployments

## License

MIT
