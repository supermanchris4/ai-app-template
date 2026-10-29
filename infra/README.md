# Infrastructure

- `bootstrap/`: the one-off script that creates the GitHub OIDC provider, the Terraform deploy and plan roles, the state bucket and, optionally, a minimal network in an AWS account. An operator runs it with their own single sign-on credentials.
- `aws/`: the Terraform for the deployed application, one file per concern, deployed into a network supplied as inputs.
- `local/`: Docker Compose for running PostgreSQL and the API on a developer's machine. `.env.example` holds only non-secret settings and throwaway values for local containers.
