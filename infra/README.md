# Infrastructure

- `bootstrap/`: the one-off script that creates the GitHub OIDC provider, the Terraform deploy and plan roles, the state bucket and, optionally, a minimal network in an AWS account. An operator runs it with their own single sign-on credentials.
- `aws/`: the Terraform for the deployed application, one file per concern, deployed into a network supplied as inputs.
- `local/`: Docker Compose for running PostgreSQL, the platform's mock identity provider image and the API on a developer's machine. The application supplies `mock-idp.json`, its roles and synthetic people. `.env.example` holds only non-secret settings and throwaway values for local containers.

The bootstrap and the application's Terraform are written by the application from [specs/12-aws-deployment.md](../specs/12-aws-deployment.md) until the platform releases them as reusable modules (the standard's work plan item F03.3).
