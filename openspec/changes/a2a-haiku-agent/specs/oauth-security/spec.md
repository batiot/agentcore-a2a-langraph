## ADDED Requirements

### Requirement: EntraID OIDC authorizer on AgentCore Runtime

The AgentCore Runtime SHALL be configured with an OAuth 2.0 authorizer that validates bearer tokens against the EntraID OIDC discovery endpoint.

#### Scenario: Authorizer validates EntraID JWT

- **WHEN** a request arrives at the AgentCore Runtime endpoint with a valid EntraID JWT bearer token
- **THEN** AgentCore SHALL validate the token using the OIDC discovery endpoint at `https://login.microsoftonline.com/5e39efe7-3ad5-4fc4-a68c-7b12fb4fdf0f/v2.0/.well-known/openid-configuration` and allow the request through

#### Scenario: Invalid or missing token is rejected

- **WHEN** a request arrives without a bearer token or with an expired/invalid token
- **THEN** AgentCore SHALL reject the request with a 401 Unauthorized response

### Requirement: Pre-generated JWT workflow

The client authentication workflow SHALL use a pre-generated JWT token stored in the `A2A_BEARER_TOKEN` environment variable, avoiding the need for an MSAL authentication flow in the client.

#### Scenario: Token sourced from environment variable

- **WHEN** the Chainlit client sends a request to the deployed AgentCore endpoint
- **THEN** it SHALL use the token from `A2A_BEARER_TOKEN` environment variable as the `Authorization: Bearer` header value

#### Scenario: Token generation documented

- **WHEN** a developer needs to generate a new JWT token
- **THEN** the project README SHALL document the steps to obtain a token from EntraID (e.g., via `curl` to the token endpoint with client credentials)

### Requirement: OAuth configuration in CloudFormation

The CloudFormation template SHALL include the OIDC authorizer configuration for the AgentCore Runtime resource, or document a post-deploy CLI step if the CloudFormation property is unavailable.

#### Scenario: OAuth configured at deploy time

- **WHEN** the CloudFormation stack is created
- **THEN** the AgentCore Runtime SHALL be configured with the EntraID OIDC authorizer, either via a CloudFormation resource property or via a custom resource that calls the AgentCore Identity API

#### Scenario: Fallback to post-deploy configuration

- **WHEN** the CloudFormation `AuthorizerConfiguration` property is not supported
- **THEN** the deploy script SHALL include a post-deploy AWS CLI step to configure the OIDC authorizer on the Runtime endpoint
