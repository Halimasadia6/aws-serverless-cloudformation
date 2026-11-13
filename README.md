# Serverless URL Shortener on AWS (CloudFormation)

A serverless URL shortener deployed entirely with AWS CloudFormation nested stacks, with automated linting and security-policy checks in GitHub Actions.

## Architecture

- Browser -> CloudFront -> private S3 bucket (static site, Origin Access Control)
- Browser -> API Gateway (HTTP API) -> Lambda (Python 3.12) -> DynamoDB
- CloudWatch dashboard and alarms -> SNS email alerts

## Stacks

| Template | What it creates |
|---|---|
| `parent.yaml` | Parent stack that deploys the four nested stacks and passes outputs between them |
| `templates/storage.yaml` | DynamoDB table (on-demand, encrypted, point-in-time recovery, TTL) |
| `templates/api.yaml` | Lambda function, least-privilege IAM role, HTTP API, CORS, throttling, X-Ray tracing |
| `templates/frontend.yaml` | Private S3 bucket and CloudFront distribution (HTTPS only, Origin Access Control) |
| `templates/monitoring.yaml` | CloudWatch dashboard, three alarms, SNS topic with email subscription |

## Security

- S3 bucket is private; only CloudFront can read it
- DynamoDB encrypted at rest with point-in-time recovery
- Lambda role can only PutItem and GetItem on one table
- HTTPS enforced by CloudFront; API throttled
- Log retention set; X-Ray tracing enabled
- Policy rules enforced by cfn-guard (`rules/security.guard`)

## Quality gates

- `cfn-lint` validates every template
- `cfn-guard` fails the build if a security rule is broken
- Change sets preview updates before they are applied
- Drift detection catches manual changes (tested by editing Lambda memory by hand)
- GitHub Actions runs lint and policy checks on every push and pull request

## Deploy

    aws cloudformation package --template-file parent.yaml --s3-bucket <artifacts-bucket> --output-template-file build/parent-packaged.yaml
    aws cloudformation deploy --template-file build/parent-packaged.yaml --stack-name urlshort-dev --parameter-overrides Environment=dev AlertEmail=you@example.com --capabilities CAPABILITY_IAM

Then insert the API URL into `src/frontend/index.html` (replace `__API_URL__`) and upload it to the site bucket shown in the stack outputs.

## Clean up

    aws s3 rm s3://<site-bucket> --recursive
    aws cloudformation delete-stack --stack-name urlshort-dev

## Notes and limitations

- Deployment is a documented CLI step. The AWS account used for this project sits under an organization policy that blocks IAM OIDC providers, so the pipeline does not deploy. It lints and policy-checks every change.
- Lambda code is inline in the template for simplicity.

## Next steps

- OIDC-based automatic deploy in an account that allows it
- Custom domain with ACM and Route 53
- Cognito authentication for link creation
- Lambda code packaged from `src/` and unit tests
- AWS WAF in front of CloudFront
