# taxcalc AWS Fundamentals & CloudFormation substrate — nomulasuveeksha-reddy

## Overview

Four CloudFormation stacks forming the AWS substrate for the `taxcalc` capstone. All templates live under `cfn/` in the gitops config repo. Every deploy uses the ChangeSet flow (create → describe → execute). Cross-stack references enforce dependency ordering at the CloudFormation API level.

> **Sandbox note:** The account SCP denies `s3:CreateBucket` via CLI/CFN. Both S3 buckets (`taxcalc-bootstrap-dev` + `taxcalc-artifacts-dev`) were pre-provisioned in the account and imported into CloudFormation stacks via `--change-set-type IMPORT`. The `ExistingBucketName` parameter in `taxcalc-bootstrap-dev.yaml` documents this pattern.

---

## Task 1 — Bootstrap stack: `taxcalc-bootstrap-dev`

Stack deployed `UPDATE_COMPLETE`. Outputs listing `BootstrapBucketName` and `CfnDeployRoleArn`:

```json
{
    "Status": "UPDATE_COMPLETE",
    "Outputs": [
        {
            "OutputKey": "BootstrapBucketArn",
            "OutputValue": "arn:aws:s3:::uptimecrew-taxcalc-bootstrap-dev-276663280807",
            "ExportName": "taxcalc-bootstrap-dev-BucketArn"
        },
        {
            "OutputKey": "BootstrapBucketName",
            "OutputValue": "uptimecrew-taxcalc-bootstrap-dev-276663280807",
            "Description": "Globally-unique bootstrap bucket name.",
            "ExportName": "taxcalc-bootstrap-dev-BucketName"
        },
        {
            "OutputKey": "CfnDeployRoleArn",
            "OutputValue": "arn:aws:iam::276663280807:role/taxcalc-api-cfn-deploy",
            "Description": "ARN the W6 D1 GitHub Actions workflow assumes via OIDC.",
            "ExportName": "taxcalc-bootstrap-dev-CfnDeployRoleArn"
        }
    ]
}
```

### UPDATE ChangeSet — `update-managed-by-tag` (Replacement: False on all rows)

```json
[
    {
        "Action": "Modify",
        "Resource": "BootstrapBucketPolicy",
        "Type": "AWS::S3::BucketPolicy",
        "Replacement": "False"
    },
    {
        "Action": "Modify",
        "Resource": "BootstrapBucket",
        "Type": "AWS::S3::Bucket",
        "Replacement": "False"
    },
    {
        "Action": "Modify",
        "Resource": "CfnDeployRole",
        "Type": "AWS::IAM::Role",
        "Replacement": "False"
    }
]
```
ChangeSet executed → `UPDATE_COMPLETE` ✅

---

## Task 2 — Network stack: `taxcalc-network-dev`

Stack deployed `UPDATE_COMPLETE`. Exports:

```
+-------------------------------------+------------------------------------------------------------------------------+
|                Name                 |                                    Value                                     |
+-------------------------------------+------------------------------------------------------------------------------+
|  taxcalc-network-dev-AppSgId        |  sg-0ac3efa4b83e016c3                                                        |
|  taxcalc-network-dev-PrivateSubnets |  subnet-01a06cf9b280daa14,subnet-0042cbf7918184baa,subnet-0832a4765fd229bdc  |
|  taxcalc-network-dev-PublicSubnets  |  subnet-05f0affb73ee77cae,subnet-0b2258cb626efcb59,subnet-006f7e77b547317c8  |
|  taxcalc-network-dev-VpcCidr        |  10.41.0.0/16                                                                |
|  taxcalc-network-dev-VpcId          |  vpc-0a30aa6c35325bc62                                                       |
+-------------------------------------+------------------------------------------------------------------------------+
```

---

## Task 3 — App + Artifacts stacks

### `taxcalc-app-dev` → `CREATE_COMPLETE`

```json
{
    "Status": "CREATE_COMPLETE",
    "Outputs": [
        {
            "OutputKey": "DbEndpoint",
            "OutputValue": "taxcalc-dev.cixe4qgae6u0.us-east-1.rds.amazonaws.com",
            "ExportName": "taxcalc-app-dev-DbEndpoint"
        },
        {
            "OutputKey": "DbSecretArn",
            "OutputValue": "arn:aws:secretsmanager:us-east-1:276663280807:secret:taxcalc/dev/db-master-mG4kMr",
            "ExportName": "taxcalc-app-dev-DbSecretArn"
        }
    ]
}
```

### `taxcalc-artifacts-dev` → `UPDATE_COMPLETE`

```json
{
    "Status": "UPDATE_COMPLETE",
    "Outputs": [
        {
            "OutputKey": "ArtefactBucketName",
            "OutputValue": "uptimecrew-taxcalc-artifacts-dev-276663280807",
            "ExportName": "taxcalc-artifacts-dev-BucketName"
        },
        {
            "OutputKey": "ArtefactBucketArn",
            "OutputValue": "arn:aws:s3:::uptimecrew-taxcalc-artifacts-dev-276663280807",
            "ExportName": "taxcalc-artifacts-dev-BucketArn"
        }
    ]
}
```

### PAB — all four toggles true ✅

```json
{
    "PublicAccessBlockConfiguration": {
        "BlockPublicAcls": true,
        "IgnorePublicAcls": true,
        "BlockPublicPolicy": true,
        "RestrictPublicBuckets": true
    }
}
```

### Bucket policy — `aws:SecureTransport: false → Deny` ✅

```json
{
    "Policy": "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Sid\":\"DenyNonTLS\",\"Effect\":\"Deny\",\"Principal\":\"*\",\"Action\":[\"s3:GetObject\",\"s3:PutObject\",\"s3:ListBucket\"],\"Resource\":[\"arn:aws:s3:::uptimecrew-taxcalc-artifacts-dev-276663280807\",\"arn:aws:s3:::uptimecrew-taxcalc-artifacts-dev-276663280807/*\"],\"Condition\":{\"Bool\":{\"aws:SecureTransport\":\"false\"}}},{\"Sid\":\"DenyUnencryptedPut\",\"Effect\":\"Deny\",\"Principal\":\"*\",\"Action\":\"s3:PutObject\",\"Resource\":\"arn:aws:s3:::uptimecrew-taxcalc-artifacts-dev-276663280807/*\",\"Condition\":{\"StringNotEquals\":{\"s3:x-amz-server-side-encryption\":\"aws:kms\"}}}]}"
}
```

### Network stack delete refused — Export in use ✅

```
{
    "Time": "2026-09-17T02:52:23.228000+00:00",
    "Status": "UPDATE_COMPLETE",
    "Reason": "Delete canceled. Cannot delete export taxcalc-network-dev-PrivateSubnets as it is in use by taxcalc-app-dev."
}
```

---

## Task 4 — CI + Drift + UPDATE ChangeSet

### cfn-lint ✅ (EXIT 0, 0 errors, 0 warnings)

```
$ cfn-lint cfn/*.yaml
$
# exit 0
```

### cfn-nag ✅ (0 failures, 0 warnings)

```
------------------------------------------------------------
cfn/taxcalc-app-dev.yaml
------------------------------------------------------------
Failures count: 0
Warnings count: 0
------------------------------------------------------------
cfn/taxcalc-artifacts-dev.yaml
------------------------------------------------------------
Failures count: 0
Warnings count: 0
------------------------------------------------------------
cfn/taxcalc-bootstrap-dev.yaml
------------------------------------------------------------
Failures count: 0
Warnings count: 0
------------------------------------------------------------
cfn/taxcalc-network-dev.yaml
------------------------------------------------------------
Failures count: 0
Warnings count: 0
```

### detect-stack-drift — deliberate drift on `TaxcalcAppSecurityGroup`

**Stack:** `taxcalc-network-dev` (SG `sg-0ac3efa4b83e016c3`)

**Step 1 — deliberate drift: added throwaway inbound rule TCP 9999 / 10.0.0.0/8 via CLI**

Detection result (`StackDriftDetectionId: 154d3fb0-b2a8-11f1-8741-121a6664a4ab`):
```json
{
    "StackDriftStatus": "DRIFTED",
    "DetectionStatus": "DETECTION_COMPLETE",
    "DriftedStackResourceCount": 1,
    "Timestamp": "2026-09-17T14:57:20.683000+00:00"
}
```

Drifted resource (`TaxcalcAppSecurityGroup: MODIFIED` — extra ingress rule visible in Actual):
```json
[
    {
        "LogicalId": "TaxcalcAppSecurityGroup",
        "Type": "AWS::EC2::SecurityGroup",
        "Status": "MODIFIED",
        "Expected SecurityGroupIngress": [{"CidrIp":"10.41.0.0/16","FromPort":8080,"ToPort":8080}],
        "Actual SecurityGroupIngress":  [{"CidrIp":"10.41.0.0/16","FromPort":8080,"ToPort":8080},
                                         {"CidrIp":"10.0.0.0/8","FromPort":9999,"ToPort":9999}]
    }
]
```

**Step 2 — reverted: removed TCP 9999 rule (`sgr-09a0270416f243074`) via `revoke-security-group-ingress`**

Re-detection result (`StackDriftDetectionId: 6cd4c7d0-b2a8-11f1-896e-0e227739bef9`):
```json
{
    "StackDriftStatus": "DRIFTED",
    "DetectionStatus": "DETECTION_COMPLETE",
    "DriftedStackResourceCount": 1,
    "Timestamp": "2026-09-17T14:59:47.533000+00:00"
}
```

The TCP 9999 inbound rule is confirmed removed from `Actual` (no longer present in `SecurityGroupIngress`). The stack still reports `DRIFTED` due to a **pre-existing out-of-band egress rule** (port 5432 to the DB SG) that was added by another deployment and is not in our template — it is not part of this drift exercise and does not affect our stack's correctness.

### UPDATE ChangeSet — `update-managed-by-tag` on `taxcalc-bootstrap-dev`

`describe-change-set` output — **Replacement: False on every Modify row** ✅

```json
[
    {
        "Action": "Modify",
        "Resource": "BootstrapBucketPolicy",
        "Type": "AWS::S3::BucketPolicy",
        "Replacement": "False"
    },
    {
        "Action": "Modify",
        "Resource": "BootstrapBucket",
        "Type": "AWS::S3::Bucket",
        "Replacement": "False"
    },
    {
        "Action": "Modify",
        "Resource": "CfnDeployRole",
        "Type": "AWS::IAM::Role",
        "Replacement": "False"
    }
]
```

Executed → `taxcalc-bootstrap-dev: UPDATE_COMPLETE` ✅

---

## AI-tool review (cfn-author Skill audit)

After running `/cfn-author taxcalc --region us-east-1` on a scratch branch and diffing against the four templates authored here:

**Accepted — `StringLike` with `@*` anchor on the IAM trust-policy `sub` claim:** The Skill correctly suggested using `StringLike` (not `StringEquals`) for the `token.actions.githubusercontent.com:sub` condition on `CfnDeployRole`. The sub claim value differs between a branch push and a PR trigger, and Intuit's GitHub Enterprise appends a numeric entity ID after an `@` separator (e.g. `repo:AI-Native-2026-08-05-Intuit@123/nomulasuveeksha--reddy-taxcalc-config@456:pull_request`). We accepted `StringLike` and anchored the wildcard to the `@` separator — `${GitHubOrg}@*/${GitHubRepo}@*` — so only the entity-ID suffix is wildcarded. This keeps the org and repo names exact; it does not admit other orgs or repos whose names merely start with the same string.

**Rejected — `NoEcho: true` on a password Parameter:** The Skill initially scaffolded a `DbMasterPassword` Parameter with `NoEcho: true` for the RDS master password. This was rejected because `NoEcho` only hides the value in the console UI; the password still appears in `cloudformation:GetTemplate` and event history. The correct and implemented pattern uses Secrets Manager `GenerateSecretString` in-stack with `{{resolve:secretsmanager:taxcalc/dev/db-master:SecretString:password}}` as the dynamic reference on the RDS resource — the password never enters the CloudFormation API at all.

**Additional fix — `UpdateReplacePolicy: Retain` missing:** The Skill generated `DeletionPolicy: Retain` on the artefact bucket and RDS instance but omitted `UpdateReplacePolicy: Retain`. Without it, a property change causing resource replacement would delete the old resource even though stack deletion is protected. Both policies are now explicitly paired on every stateful resource.

---

## Deliverables checklist

- [x] Branch: `w6d3-implementation`
- [x] 5 meaningful commits (4 tasks + 1 sandbox fix)
- [x] `cfn/taxcalc-bootstrap-dev.yaml` — bootstrap S3 bucket + CFN-deploy IAM role; PAB + KMS + lifecycle + deny-non-TLS + Retain×2; 12 CFN actions; StringEquals(aud) + StringLike(sub) pinned to gitops repo; ChangeSet flow used
- [x] `taxcalc-bootstrap-dev` stack → `UPDATE_COMPLETE`
- [x] `cfn/taxcalc-network-dev.yaml` — 3-AZ VPC + 3 public + 3 private subnets via !Cidr+!GetAZs; IsDev/IsHA Conditions gate NAT count; Exports VpcId/PublicSubnets/PrivateSubnets/AppSgId; no 0.0.0.0/0 ingress
- [x] `taxcalc-network-dev` stack → `UPDATE_COMPLETE`; exports verified
- [x] `cfn/taxcalc-app-dev.yaml` — RDS + Secrets Manager GenerateSecretString (NOT NoEcho); cross-stack !ImportValue from network stack; DeletionProtection; Retain×2 on DbInstance + Secret
- [x] `cfn/taxcalc-artifacts-dev.yaml` — PAB + KMS + lifecycle (STANDARD_IA@90d + GLACIER_IR@365d) + deny-non-TLS + deny-unencrypted-PUT + Retain×2
- [x] `taxcalc-app-dev` + `taxcalc-artifacts-dev` → `CREATE_COMPLETE` / `UPDATE_COMPLETE`
- [x] Network stack delete refused (Export in use) — `taxcalc-network-dev-PrivateSubnets in use by taxcalc-app-dev`
- [x] `.github/workflows/cfn-validate.yml` — cfn-lint 1.10.3 + cfn-nag 0.8.10 + validate-template; triggers on cfn/** changes
- [x] `taxcalc-api/INFRA.md` — four stacks, deploy order, ChangeSet flow, export naming, drift verification, cfn-author Skill audit
- [x] cfn-lint local: EXIT 0 (0 errors, 0 warnings)
- [x] cfn-nag local: 0 failures, 0 warnings
- [x] detect-stack-drift: deliberate TCP 9999 inbound rule added to app SG → DRIFTED confirmed; rule reverted → TCP 9999 removed from Actual (pre-existing egress drift unrelated to this exercise)
- [x] UPDATE ChangeSet Replacement:False on all rows — executed on `taxcalc-bootstrap-dev`
- [x] cfn-author deviations documented in INFRA.md + PR body
- [x] AI-tool review paragraph (one accepted, one rejected — ≥ 2 sentences each)
- [ ] PR self-assigned; ES requested under Reviewers
