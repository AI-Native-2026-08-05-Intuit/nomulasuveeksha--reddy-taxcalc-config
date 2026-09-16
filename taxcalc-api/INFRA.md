# taxcalc-api/INFRA.md

How this service's AWS substrate is provisioned and what deviated from the cfn-author Skill's defaults.

## Stack layout (after today)

| Stack | File | Purpose |
|-------|------|---------|
| `taxcalc-bootstrap-dev` | `cfn/taxcalc-bootstrap-dev.yaml` | Hardened S3 bootstrap bucket (KMS, PAB, lifecycle, deny-non-TLS, Retain×2) + `taxcalc-api-cfn-deploy` IAM role for GitHub Actions OIDC. First stack deployed; independent of all others. |
| `taxcalc-network-dev` | `cfn/taxcalc-network-dev.yaml` | 3-AZ VPC (10.41.0.0/16), 3 public + 3 private subnets via `!Cidr`+`!GetAZs`, IGW, 1–3 NAT GWs (Conditions-gated by EnvName: single in dev, per-AZ in staging/prod), application security group. Exports `VpcId`, `VpcCidr`, `PrivateSubnets`, `PublicSubnets`, `AppSgId`. |
| `taxcalc-artifacts-dev` | `cfn/taxcalc-artifacts-dev.yaml` | Hardened S3 artefact bucket (KMS, PAB, lifecycle to STANDARD_IA@90d → GLACIER_IR@365d, deny-non-TLS, deny-unencrypted-PUT, Retain×2). Independent of the network stack; deployed before the app stack. |
| `taxcalc-app-dev` | `cfn/taxcalc-app-dev.yaml` | RDS PostgreSQL 16.3 + DB SG + DB subnet group + Secrets Manager master credentials (in-stack via `GenerateSecretString`). Consumes network stack exports via `!ImportValue`. DB master password lives exclusively in Secrets Manager — NEVER passed as a `NoEcho` Parameter. |

## Deploy order

Deploy in this sequence to satisfy cross-stack dependencies:

```
1. taxcalc-bootstrap-dev   (Task 1)  — bootstrap bucket + CFN-deploy IAM role
2. taxcalc-artifacts-dev   (Task 3)  — artefact bucket; independent of network
3. taxcalc-network-dev     (Task 2)  — VPC + subnets + SG; exports consumed by app
4. taxcalc-app-dev         (Task 3)  — RDS + secret; imports from taxcalc-network-dev
```

## ChangeSet flow (every stack, every change)

Every deploy — initial create and every subsequent update — uses the ChangeSet flow:

```bash
# 1. Create the ChangeSet (--change-set-type CREATE for new stacks, UPDATE for existing)
aws cloudformation create-change-set \
  --stack-name <STACK_NAME> \
  --change-set-name <CHANGESET_NAME> \
  --change-set-type CREATE \
  --capabilities CAPABILITY_NAMED_IAM \
  --template-body file://cfn/<FILE>.yaml \
  --parameters ParameterKey=EnvName,ParameterValue=dev \
  --region us-east-1

# 2. Review the diff — paste into PR body before executing
aws cloudformation describe-change-set \
  --stack-name <STACK_NAME> \
  --change-set-name <CHANGESET_NAME> \
  --region us-east-1

# 3. Execute only after review
aws cloudformation execute-change-set \
  --stack-name <STACK_NAME> \
  --change-set-name <CHANGESET_NAME> \
  --region us-east-1

# 4. Wait for completion
aws cloudformation wait stack-create-complete \
  --stack-name <STACK_NAME> \
  --region us-east-1
```

The `describe-change-set` output enumerates every `Replace`, `Modify`, `Add`, and `Remove`. PR reviewers read the diff before the `execute` step. The bootstrap stack's `CfnDeployRole` is what GitHub Actions assumes via OIDC to run these commands.

## Cross-stack export naming

Export names follow the pattern `<StackName>-<Resource>` using `!Sub "${AWS::StackName}-<Resource>"`:

| Export | Value |
|--------|-------|
| `taxcalc-network-dev-VpcId` | VPC logical ID resolved to `vpc-xxxx` |
| `taxcalc-network-dev-VpcCidr` | `10.41.0.0/16` |
| `taxcalc-network-dev-PrivateSubnets` | Comma-joined list of 3 private subnet IDs |
| `taxcalc-network-dev-PublicSubnets` | Comma-joined list of 3 public subnet IDs |
| `taxcalc-network-dev-AppSgId` | App security group ID |
| `taxcalc-bootstrap-dev-BucketName` | Bootstrap bucket name |
| `taxcalc-bootstrap-dev-CfnDeployRoleArn` | IAM role ARN for OIDC |
| `taxcalc-artifacts-dev-BucketName` | Artefact bucket name |

Consumers import with `!ImportValue "taxcalc-network-dev-PrivateSubnets"` then `!Split [",", ...]` to recover a list. CFN enforces cross-stack safety: attempting to delete `taxcalc-network-dev` while `taxcalc-app-dev` imports its exports will be refused with an "Export ... is in use by stack ..." error.

## Drift detection (deliberate verification)

To verify drift detection works:

```bash
# 1. Deliberately drift a resource in the AWS console (e.g. add a stray tag to the bucket)

# 2. Trigger drift detection
aws cloudformation detect-stack-drift \
  --stack-name taxcalc-artifacts-dev \
  --region us-east-1
# → returns: {"StackDriftDetectionId": "..."}

# 3. Poll until complete
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id <ID> \
  --region us-east-1

# 4. List drifted resources
aws cloudformation describe-stack-resource-drifts \
  --stack-name taxcalc-artifacts-dev \
  --region us-east-1 \
  --query "StackResourceDrifts[?StackResourceDriftStatus!='IN_SYNC']"

# 5. Revert: re-deploy the original template via ChangeSet → status returns IN_SYNC
```

For Task 4 verification, the `TaxcalcArtifactsBucket` resource was deliberately drifted (a stray tag added via the AWS console), confirmed `DRIFTED`/`MODIFIED` in the drift report, then reverted by re-running the ChangeSet on the original template.

## cfn-author Skill audit notes

After running `/cfn-author taxcalc --region us-east-1` against a scratch branch and comparing its output to the four templates authored here, the following deviations were identified:

**Suggestion accepted — StringLike on the IAM trust-policy `sub` claim:** The cfn-author Skill correctly suggested using `StringLike` (not `StringEquals`) for the `token.actions.githubusercontent.com:sub` condition on the `CfnDeployRole` trust policy. This is correct because the sub claim can be any of `repo:org/repo:ref:refs/heads/main` or `repo:org/repo:pull_request`, and the wildcard allows the pattern to match both. We accepted this and pinned the prefix exactly to `repo:${GitHubOrg}/${GitHubRepo}:*` with the Skill's guidance.

**Suggestion rejected — `NoEcho: true` on a password Parameter:** The cfn-author Skill initially scaffolded a `DbMasterPassword` Parameter with `NoEcho: true` for the RDS master password. This was rejected because `NoEcho` only hides the value in the console; the password still appears in `cloudformation:GetTemplate` output and in the stack's event history. The correct pattern — and what is implemented here — is a Secrets Manager `GenerateSecretString` block inside the `DbMasterSecret` resource, with the RDS instance consuming the secret via the `{{resolve:secretsmanager:taxcalc/dev/db-master:SecretString:password}}` dynamic reference. This way, the password value never enters the CloudFormation API at all.

**Additional fix — `UpdateReplacePolicy: Retain` paired with `DeletionPolicy: Retain`:** The Skill generated `DeletionPolicy: Retain` on the artefact bucket and the DB instance but omitted `UpdateReplacePolicy: Retain`. Without `UpdateReplacePolicy: Retain`, a CloudFormation update that causes resource replacement (e.g. renaming the bucket) would delete the old resource even though `DeletionPolicy` would protect it on stack deletion. Both policies are now present on every stateful resource (`BootstrapBucket`, `TaxcalcArtifactsBucket`, `DbMasterSecret`, `DbInstance`).
