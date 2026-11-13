# Infrastructure

- `bootstrap/`: the one-off Terraform root, and the script that applies it, that creates the GitHub OIDC provider, the Terraform deploy and plan roles, the state bucket and, optionally, a minimal network in an AWS account. An operator runs it with their own single sign-on credentials.
- `aws/`: the Terraform root for the deployed application, one file per concern, deployed into a network supplied as inputs.
- `local/`: Docker Compose for running PostgreSQL, the platform's mock identity provider image and the API on a developer's machine. The application supplies `mock-idp.json`, its roles and synthetic people. `.env.example` holds only non-secret settings and throwaway values for local containers.

Both roots compose the platform's Terraform modules from `ai-app-platform`, at the commit a release tag names, and hold only the application's own inputs, names, outputs and resources ([specs/12-aws-deployment.md](../specs/12-aws-deployment.md#platform-modules)).
