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

For Task 4 verification, `detect-stack-drift` was run on `taxcalc-artifacts-dev` and returned `IN_SYNC`. The deliberate-drift exercise (adding a stray console tag, confirming DRIFTED, then reverting) was not completed: the sandbox SCP blocks `s3:PutBucketTagging` via CLI, and the bucket's system tags (`aws:cloudformation:*`) prevent full tag replacement, so the stray tag could not be added out-of-band during this session. The drift commands and their expected output are documented above for completeness.

## cfn-author Skill audit notes

These templates were scaffolded and iterated with Cursor (cursoragent@cursor.com). The following deviations from the initial scaffold were identified and either accepted or rejected:

**Suggestion accepted — `StringLike` with `@*` anchor on the OIDC trust-policy `sub` claim:**
The initial scaffold used `StringEquals` on the `sub` claim, which fails in this Intuit GitHub Enterprise account because the enterprise appends a numeric entity ID after an `@` separator (e.g. `repo:AI-Native-2026-08-05-Intuit@123/nomulasuveeksha--reddy-taxcalc-config@456:pull_request` instead of the plain `repo:org/repo:pull_request`). We switched to `StringLike` and anchored the wildcard to the `@` separator — `${GitHubOrg}@*/${GitHubRepo}@*` — so only the entity-ID suffix is wildcarded. This keeps the match scoped to this exact org and repo name; it does not admit other orgs or repos whose names merely start with the same string.

**Suggestion rejected — `NoEcho: true` on a password Parameter:**
The initial scaffold created a `DbMasterPassword` Parameter with `NoEcho: true` for the RDS master password. This was rejected because `NoEcho` only suppresses the value in the console UI; the password is still accessible via `cloudformation:GetTemplate` and visible in stack event history. The implemented pattern uses Secrets Manager `GenerateSecretString` in-stack so CloudFormation generates and stores the password directly in Secrets Manager — it never appears in CFN API responses. The RDS instance resolves it at deploy time via `{{resolve:secretsmanager:taxcalc/${EnvName}/db-master:SecretString:password}}`, using the parameterised `EnvName` so staging and prod stacks each read their own environment's secret.

**Additional fix — `UpdateReplacePolicy: Retain` missing alongside `DeletionPolicy: Retain`:**
The initial scaffold applied `DeletionPolicy: Retain` to stateful resources but omitted `UpdateReplacePolicy: Retain`. Without it, a CloudFormation update that triggers resource replacement (e.g. a bucket rename) silently deletes the old resource even though the stack-deletion policy would have protected it. Both policies now travel together on every stateful resource: `BootstrapBucket`, `TaxcalcArtifactsBucket`, `DbMasterSecret`, and `DbInstance`.
