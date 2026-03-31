## ADDED Requirements

### Requirement: CloudFormation deployment stack

The system SHALL provide a single CloudFormation template (`infra/template.yaml`) that deploys all required AWS resources for the haiku agent on AgentCore Runtime.

#### Scenario: Stack creates all required resources

- **WHEN** the CloudFormation stack is created
- **THEN** it SHALL provision: an ECR repository, an S3 bucket for source artifacts, a CodeBuild project (ARM64), IAM roles (agent execution, CodeBuild, Lambda custom resource), a Lambda function to trigger CodeBuild, a CloudWatch log group, and an `AWS::BedrockAgentCore::Runtime` resource

#### Scenario: Stack uses eu-west-3 region

- **WHEN** the stack is deployed
- **THEN** all resources SHALL be created in the `eu-west-3` region

### Requirement: CodeBuild Docker image build

The CloudFormation template SHALL include a CodeBuild project that builds an ARM64 Docker image from the agent source code (sourced from S3) and pushes it to the ECR repository. The build uses a `buildspec.yml` file from the source bundle and the canonical `Dockerfile` from the repository.

#### Scenario: CodeBuild produces a working container image

- **WHEN** CodeBuild runs during stack creation
- **THEN** it SHALL pull the source bundle from S3, build a Docker image using the repository's `Dockerfile` and `buildspec.yml`, push it to the ECR repository, and the image SHALL be runnable on AgentCore Runtime listening on port 9000

#### Scenario: CodeBuild uses ARM64 architecture

- **WHEN** the CodeBuild project runs
- **THEN** it SHALL use the `aws/codebuild/amazonlinux2-aarch64-standard:3.0` image with `ARM_CONTAINER` compute type

### Requirement: AgentCore Runtime A2A configuration

The `AWS::BedrockAgentCore::Runtime` resource SHALL be configured with protocol type `A2A`, port `9000`, and root path `/`.

#### Scenario: Runtime serves A2A protocol

- **WHEN** the AgentCore Runtime is created
- **THEN** it SHALL be configured to proxy A2A JSON-RPC requests to the container on port 9000 with root path `/`

### Requirement: Deploy and destroy scripts

The system SHALL provide shell scripts for one-command deployment (`infra/deploy.sh`) and teardown (`infra/destroy.sh`).

#### Scenario: Deploy script creates the stack

- **WHEN** the user runs `infra/deploy.sh`
- **THEN** it SHALL package the project source into a zip, upload it to the S3 source bucket, create the CloudFormation stack, wait for completion, and output the AgentCore Runtime endpoint URL

#### Scenario: Deploy script configures OAuth fallback

- **WHEN** the CloudFormation stack completes and the `AuthorizerConfiguration` was not applied for A2A protocol
- **THEN** `deploy.sh` SHALL run AWS CLI commands to configure the OIDC authorizer on the Runtime endpoint

#### Scenario: Destroy script cleans up all resources

- **WHEN** the user runs `infra/destroy.sh`
- **THEN** it SHALL delete all ECR images, delete S3 source artifacts, delete the CloudFormation stack, and wait for deletion to complete

### Requirement: IAM least-privilege roles

The CloudFormation template SHALL define IAM roles with minimal permissions required for each service.

#### Scenario: Agent execution role has Bedrock access

- **WHEN** the agent execution IAM role is created
- **THEN** it SHALL grant `bedrock:InvokeModelWithResponseStream` and `bedrock:InvokeModel` permissions for the Nova Light model in eu-west-3, and basic CloudWatch Logs permissions

#### Scenario: CodeBuild role has ECR, S3, and logs access

- **WHEN** the CodeBuild IAM role is created
- **THEN** it SHALL grant ECR push/pull permissions scoped to the created repository, S3 read permissions scoped to the source bucket, and CloudWatch Logs write permissions
