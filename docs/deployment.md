# AWS deployment

The supplied infrastructure and workflow have been inspected but not deployed. These instructions describe their configuration sequence; they are not evidence of a successful deployment. Resources and model calls incur charges.

## Prepare the backend

Authenticate the AWS CLI in the intended account and region. The defaults consistently use `us-east-1` and the resource prefix `research-agent`.

The S3 bucket name in both bootstrap scripts and `terraform/main.tf` is `research-agent-tfstate`. Choose a globally unique bucket name and replace it consistently before running a bootstrap script. Keep the DynamoDB table and region consistent with the backend block too. The scripts can misreport AWS errors as an existing resource; verify the resulting bucket and table.

```bash
bash bootstrap.sh us-east-1
terraform -chdir=terraform init
terraform -chdir=terraform validate
```

On Windows, use `bootstrap.bat`. These commands create or access infrastructure, so they were not run as part of repository publication.

## Provision infrastructure and images

The required Terraform inputs are `app_image` and `pyrit_image`; `tensorzero_image` defaults to `placeholder` but needs a real image to run. All three Docker builds use the repository root as their context.

For initial setup, the original infrastructure supports provisioning with placeholder image strings:

```bash
terraform -chdir=terraform plan \
  -var='app_image=placeholder' \
  -var='pyrit_image=placeholder' \
  -var='tensorzero_image=placeholder'
terraform -chdir=terraform apply \
  -var='app_image=placeholder' \
  -var='pyrit_image=placeholder' \
  -var='tensorzero_image=placeholder'
```

Review the plan before applying. Set `TF_VAR_api_key` privately for the shared API key, and optionally supply `acm_certificate_arn` for HTTPS on the application listener. Placeholder images cannot start: this is only an infrastructure bootstrap, not a working app. The dashboard service always requests one task, even if `app_desired_count` is zero.

Terraform defines the VPC, public ECS tasks, private RDS/Redis subnets, ALB, ECR repositories, IAM roles, guardrail, secret, logging, autoscaling, and a scheduled ECS task. It does not build images.

Retrieve the outputs with:

```bash
terraform -chdir=terraform output
```

Edit `research-agent/config` in Secrets Manager to replace the OpenAI/Groq placeholders, set the API key, and either fill in or clear the LangSmith key. Preserve the generated database and Redis addresses and guardrail identifiers. The API caches the secret in-process, and ECS resolves sidecar secrets at task startup; restart tasks after changes.

Terraform manages this secret's JSON, so subsequent applies can overwrite manual edits. Review plans for secret changes and avoid putting actual credentials in tracked Terraform files. State contains sensitive values and must remain protected.

## Build and deploy with GitHub Actions

The [deployment workflow](../.github/workflows/deploy.yml) uses `workflow_dispatch` so publishing this repository does not automatically deploy AWS resources.

1. Provision the expected AWS resources and populate the secret first.
2. Configure repository Actions secrets `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` for the existing workflow. It does not configure an OIDC role despite declaring `id-token: write`, and it does not pass an AWS session token.
3. Run **Build and Deploy** manually from the Actions tab, or use:

   ```bash
   gh workflow run deploy.yml --ref main
   ```

The workflow builds three images, pushes commit and `latest` tags to ECR, registers ECS task revisions, and updates services. It waits for the app service and includes rollback steps, but does not independently wait for the dashboard service to stabilize.

It hardcodes cluster/service/ECR names beginning with `research-agent` and the `us-east-1` region. Changing Terraform's project or region also requires updating the workflow, bootstrap backend, and fixed configuration references.

For subsequent Terraform runs, supply the real image URIs from ECR instead of the placeholders. Terraform and GitHub Actions both manage task definitions; review plans so an apply does not revert a workflow deployment.

## Access and operational caveats

- Application: `http://ALB_DNS/`, or the configured HTTPS domain when using an ACM certificate.
- Dashboard: `http://ALB_DNS:8001/`; this listener has no authentication or TLS in the supplied configuration. Limit access before exposing it.
- The dashboard target does not include `X-API-Key`, so tests against an authenticated app do not establish guardrail effectiveness.
- The Monday 02:00 UTC EventBridge schedule starts the dashboard container. No command invokes `/run-attacks`; scheduled red-team execution is not implemented.
- RDS deletion protection is **false**. A final snapshot is requested, but the fixed snapshot identifier can conflict with an existing snapshot. ECR repositories set `force_delete = true`.
- Bootstrap bucket/table resources are outside this Terraform configuration. Database snapshots, backend state, and other retained resources need separate lifecycle management.

Review [the implementation limitations](../README.md#known-limitations) before operating this deployment.
