# Runbook

> **APPLICATION STUB.** Fill in the values in braces for this application, and record each step's date when it is done. Delete this callout once the first environment is set up.

The one-off setup steps for each environment of the application. The specifications they implement are named in each section.

## Identity provider registrations

Following [specs/50-auth.md](../specs/50-auth.md#app-registrations), for each environment (`dev`, `test`, `prod`):

| Step | Value | Done |
|---|---|---|
| Create the application's registration in the internal issuer, as a confidential web client | Client ID: {client ID} | |
| Set the redirect URI | `{address}/api/auth/callback` | |
| Expose the `access_as_user` scope and define the app roles in [specs/50-auth.md](../specs/50-auth.md#application-roles) | | |
| Set the registration to issue v2.0 access tokens, if the provider is Entra ID (`requestedAccessTokenVersion` set to `2`) | | |
| Create the client certificate, upload its public key to the registration, and store the certificate and private key in Secrets Manager | Secret ARN: {ARN} | |
| Assign the app roles to people, groups and consumer systems | | |
| Repeat for the external issuer, if the application has external users | Client ID: {client ID} | |
| Create a registration for each consumer system and grant it the application's app roles | {consumer}: {client ID} | |

## AWS account bootstrap

Following [specs/12-aws-deployment.md](../specs/12-aws-deployment.md#account-bootstrap), once per account, with your own single sign-on session:

| Step | Value | Done |
|---|---|---|
| Sign in: `aws sso login --profile {profile}` | Account ID: {account ID} | |
| Run the bootstrap in `infra/bootstrap/` | | |
| Record the outputs | Deploy role ARN, plan role ARN, state bucket, OIDC provider ARN, VPC and subnet IDs | |
| Request or import the certificates for the load balancer and, if the application has a custom domain, for CloudFront in `us-east-1` | Certificate ARNs | |

## GitHub

Following [specs/13-delivery-pipeline.md](../specs/13-delivery-pipeline.md):

| Step | Value | Done |
|---|---|---|
| Create the GitHub Environments `dev`, `test` and `prod` | | |
| Add a required reviewer and restrict deployment to `main` on `test` and `prod` | Reviewer: {user} | |
| Add the GitHub variables for each environment: role ARNs, region, network IDs, state bucket, certificate ARNs | | |
| Add the branch ruleset on `main`: pull request required, required checks from `ci.yml`, no force pushes | | |
| Turn on secret scanning and push protection, where the plan provides them | | |
