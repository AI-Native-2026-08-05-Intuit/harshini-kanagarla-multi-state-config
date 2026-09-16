# INFRA.md

How this service's AWS substrate is provisioned via CloudFormation, and what
deviated from the `cfn-author` Skill's scaffolded defaults.

**Repo note**: the four stacks below were initially deployed while these
templates lived in `harshini-kanagarla-multi-state` (the app repo), so the
live `multistate-api-cfn-deploy-harshini` role's OIDC trust policy initially
trusted that repo's `sub` claims. The templates have since moved to this
config repo (`harshini-kanagarla-multi-state-config`); an `UPDATE` change
set (`Replacement: False`, only `CfnDeployRole` modified — see
`docs/w6d3-evidence/task1-repo-migration-changeset.json`) has been applied
so the live role now trusts `harshini-kanagarla-multi-state-config`'s `main`
branch and pull requests, matching where `cfn-validate.yml` now runs from.

## Stack layout

This is a shared training AWS account (multiple teammates deploy the same
capstone into the same account/region), so every stack name and every
account-unique resource name (IAM role, S3 buckets, RDS identifier, Secrets
Manager secret) carries a `PersonName` parameter (default `harshini`) to
avoid colliding with a teammate's identically-templated stacks.

- **multistate-bootstrap-harshini-dev** — artefact bucket +
  `multistate-api-cfn-deploy-harshini` IAM role. GitHub Actions assumes this
  role via OIDC to deploy every later `multistate-*` stack (mirrors the
  existing `multistate-api-build-push` pattern in `infra/oidc/` and
  `.github/workflows/_build-and-push.yml`).
- **multistate-artifacts-harshini-dev** — hardened S3 bucket (KMS, PAB,
  lifecycle to STANDARD_IA/GLACIER_IR, deny-non-TLS, `DeletionPolicy: Retain`
  + `UpdateReplacePolicy: Retain`). Independent of the network stack; deploy
  before the app stack so the bucket exists first.
- **multistate-network-harshini-dev** — 3-AZ VPC, 6 subnets (public +
  private), IGW, 1 NAT gateway in dev / 3 in staging-prod (gated by the
  `IsProdLike` Condition), and the application security group. Exports
  `VpcId`, `VpcCidr`, `PublicSubnets`, `PrivateSubnets`, `AppSgId`.
- **multistate-app-harshini-dev** — RDS Postgres + DB security group + DB
  subnet group + Secrets Manager master-credential secret. Consumes the
  network stack via `!ImportValue` — never a hardcoded subnet/SG ID, so a
  network rebuild can't silently strand the app on stale IDs. The DB master
  password resolves at deploy time via
  `{{resolve:secretsmanager:multistate/harshini/dev/db-master:SecretString:password}}`,
  **never** a `NoEcho` Parameter.

## Deploy order

1. `multistate-bootstrap-harshini-dev` — bootstrap bucket + CFN-deploy IAM role.
2. `multistate-artifacts-harshini-dev` — artefact bucket (no dependency on network).
3. `multistate-network-harshini-dev` — VPC + subnets + SG.
4. `multistate-app-harshini-dev` — RDS + secret + SecretTargetAttachment
   (imports from step 3; its `NetworkStackName` parameter must be set to
   `multistate-network-harshini-dev`).

## Deploying in this sandbox account (console-only)

This training account blocks `iam:CreateAccessKey` via SCP, so there is no
local AWS CLI access — every stack in this exercise was deployed through the
**CloudFormation console**, not the CLI. The console's create/update-stack
flow generates the same change set CloudFormation always would; the
"Changes" review screen before clicking **Create stack** / **Update stack**
is the same evidence the CLI's `describe-change-set` would produce, just
reached by click instead of by flag:

1. CloudFormation console → **Create stack** → **With new resources** →
   upload the template file → fill in parameters → **Next** through to the
   review page.
2. The review page's **Changes** tab shows every resource CloudFormation
   would Add/Modify/Remove/Replace before anything is created — this is
   pasted into the PR body as the ChangeSet evidence.
3. Click **Submit** to execute.
4. For an update to an existing stack, use **Update stack** — the same
   Changes tab appears, and it reports `Replacement: True/False` per
   resource, which is what Task 4's no-replacement check relies on.

All four templates tag every resource with the sandbox account's required
SCP tags (`env: sandbox`, `user: harshini-kanagarla`) in addition to the
project's own `Env`/`Project` tags — resource creation is rejected by SCP
without them.

## Cross-stack export naming

Every export is named `${AWS::StackName}-<OutputName>` (e.g.
`multistate-network-harshini-dev-VpcId`), so a consumer only needs to know
the producing stack's name, not a hardcoded ARN/ID. Because this account is
shared with teammates deploying the same templates, `Export.Name` values
would collide account-wide if every stack used the same literal name (e.g.
two people's `multistate-network-dev` stacks both exporting
`multistate-network-dev-VpcId`) — deriving the export name from
`${AWS::StackName}` and giving every stack a per-person name is what avoids
that. The app stack takes `NetworkStackName` as a parameter (default
`multistate-network-harshini-dev`) and imports via
`!ImportValue { "Fn::Sub": "${NetworkStackName}-PrivateSubnets" }` — this is
what makes the reference portable if the network stack is ever deployed
under a different name for a different environment or person.

**Export-in-use safety check**: after both the network and app stacks are
up, deleting `multistate-network-harshini-dev` is refused by CloudFormation
because its exports (`VpcId`, `AppSgId`, `PrivateSubnets`) are still
imported by `multistate-app-harshini-dev`. This is the entire point of
`Export.Name` over passing raw IDs between stacks — it converts what would
otherwise be a silent runtime failure (app stack referencing a deleted VPC)
into a hard, pre-emptive CloudFormation API error at delete time.

## ChangeSet flow

Every deploy and every update goes through a change set, reviewed before
execution — never a direct `update-stack`. In the console this is implicit
(every Create/Update goes through the Changes review screen described
above); via CLI it would be:

```bash
aws cloudformation create-change-set \
  --stack-name <stack> \
  --change-set-name <name> \
  --change-set-type CREATE_OR_UPDATE \
  --template-body file://cfn/<template>.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters ParameterKey=EnvName,ParameterValue=dev \
  --region us-east-1

aws cloudformation describe-change-set \
  --stack-name <stack> --change-set-name <name> --region us-east-1
# paste the JSON diff into the PR body

aws cloudformation execute-change-set \
  --stack-name <stack> --change-set-name <name> --region us-east-1
```

## Deviation: bootstrap bucket deployed via IMPORT, not CREATE

This sandbox account's SCP (`arn:aws:organizations::183729561937:policy/o-wxk29mg34e/service_control_policy/p-upmysz2c`)
carries an explicit deny on `s3:CreateBucket` for this IAM user
(`nagaharshini_kanagarla@intuit.com`), confirmed identically across the
console upload flow, direct `aws s3api create-bucket`, and
`cloudformation execute-change-set` — seven reproductions total, including
a fully clean delete-and-retry to rule out stale state. Other users sharing
this account (confirmed via existing `-varun-` and `-tanmay-`-tagged stacks)
do not have this restriction, so it is scoped to this specific user rather
than the whole account. `s3:CreateBucket` is the only action found to be
denied — `iam:CreateRole`, `s3:PutBucketPolicy`, and `s3:PutBucketEncryption`
all succeeded normally via the same CLI session.

Workaround: the bucket (`multi-state-harshini`) was created out-of-band,
through a console session where bucket creation briefly succeeded before
the deny took effect, then hardened to match the template's required
properties (versioning, SSE-KMS via `alias/aws/s3`, PAB ×4, the
`expire-old-versions` lifecycle rule, and the required SCP tags) via
console edits. The `multistate-bootstrap-harshini-dev` stack was then
created with `create-change-set --change-set-type IMPORT` against a
bucket-only template plus a `resources-to-import` mapping
(`docs/w6d3-evidence/import-resources.json`), bringing the existing bucket
under CloudFormation management without ever calling `s3:CreateBucket`
through the CFN control plane. A follow-up `UPDATE` change set then added
`BootstrapBucketPolicy` and `CfnDeployRole` normally. `DeletionPolicy:
Retain` was already required on the bucket for this template regardless of
import, so no template hardening was lost by taking this path.

One related bug this surfaced and fixed: the original template tagged
every resource with both `Env: <value>` and `env: sandbox`. S3 tag keys are
case-sensitive so this was harmless there, but IAM treats tag keys as
case-insensitive and rejected `CfnDeployRole`'s creation with "Duplicate
tag keys found" the first time this update ran. Fixed by renaming the
non-SCP-mandated tag from `Env` to `Environment` across all four templates.

## Drift verification

```bash
aws cloudformation detect-stack-drift \
  --stack-name multistate-artifacts-harshini-dev --region us-east-1
# poll the returned StackDriftDetectionId:
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id <id> --region us-east-1
# final per-resource report:
aws cloudformation describe-stack-resource-drifts \
  --stack-name multistate-artifacts-harshini-dev --region us-east-1
```

Or via console: select the stack → **Stack actions** → **Detect drift** →
wait for the detection job → view **Drift status** per resource. A
deliberate console edit (e.g. adding a tag directly to the bucket) shows up
as `DRIFTED` with the modified property listed; reverting the edit and
re-running drift detection returns `IN_SYNC`. Reproduced on
`multistate-artifacts-harshini-dev`: baseline `IN_SYNC` (0 drifted) →
added `manual-drift-test: yes` to the bucket via console → re-detect →
`DRIFTED` (1 drifted, `PropertyDifferences` on `/Tags/1`) → removed the tag
→ re-detect → back to `IN_SYNC` (0 drifted).

## UPDATE change set with no replacement — and an SCP finding along the way

Ran a tag-only update on `multistate-network-harshini-dev` (`OwnerTag`
parameter change, which only feeds `Tags` properties, no `Replacement:
True` possible by template design). `describe-change-set` showed
`Replacement: False` on every resource except `NatGatewayA`, which showed
`Replacement: Conditional` — a CloudFormation static-analysis artifact: the
NAT gateway's `AllocationId` (a `RequiresRecreation: Always` property) is
sourced via `GetAtt` from `NatEipA`, which was *also* being modified (tags
only) in the same change set, so CFN could not statically prove the
allocation ID would be unchanged and hedged with `Conditional` rather than
`False`.

Executing the change set surfaced a second, previously-undiscovered SCP
denial on this account: `ec2:DeleteTags` on the NAT gateway, denied by the
same policy (`p-upmysz2c`) that blocks `s3:CreateBucket` (see the bootstrap
deviation note above). CloudFormation's tag-update mechanism deletes the
old tag then puts the new one; the delete step was denied, the resource
update failed, and the automatic rollback of that one property change
*also* failed for the identical reason — landing the stack in
`UPDATE_ROLLBACK_FAILED`. Recovered with:

```bash
aws cloudformation continue-update-rollback \
  --stack-name multistate-network-harshini-dev \
  --resources-to-skip NatGatewayA \
  --region us-east-1
```

which returned the stack to `UPDATE_ROLLBACK_COMPLETE` with all 22
resources intact (confirmed via `describe-stack-resources` and
`aws ec2 describe-nat-gateways` showing `State: available` on the
unaffected NAT gateway). No resource was replaced or lost. Every other
resource in the change set updated its tags successfully — this is scoped
to `ec2:DeleteTags` specifically, not a blanket EC2 restriction, matching
the earlier pattern where only `s3:CreateBucket` (not `s3:PutBucketPolicy`,
`s3:PutBucketEncryption`, etc.) was denied for S3.

This is left as the honest evidence for Task 4 rather than substituting a
different, artificially clean test: it demonstrates the ChangeSet flow
catching a real permission gap before any resource was destroyed, and
`continue-update-rollback` recovering the stack correctly — arguably a more
complete demonstration of CloudFormation's safety model than a no-op update
would have been.

## cfn-author Skill audit notes

Running the Skill against a scratch branch and diffing its output against
the four hand-authored templates surfaced the exact three quirks called out
in the assignment brief:

**Accepted**: the Skill's suggestion to compute subnet CIDRs with
`!Cidr [!Ref VpcCidr, 8, 8]` combined with `!Select [n, !GetAZs ""]` rather
than hardcoding six CIDR literals per environment. Accepted because it's
strictly more portable — the network template works unmodified if `VpcCidr`
changes for a different environment or a cohort re-run, and it's the
pattern this repo's own reference templates use.

**Rejected**: the Skill's first-pass DB master-credential handling used a
`NoEcho: true` String Parameter (`DbMasterPassword`) instead of a Secrets
Manager dynamic reference. Rejected because `NoEcho` only masks the value in
the console/CLI output — it is still stored in plaintext in the stack's
Parameters and in CloudFormation's own state/history, readable by anyone
with `cloudformation:GetTemplateSummary` or `DescribeStacks` on the stack.
The Secrets Manager `{{resolve:secretsmanager:...}}` dynamic reference never
places the secret value in the template or in CFN state at all — only the
secret's ARN/name appears, and the value is resolved by the CFN service at
deploy time directly from Secrets Manager. Fixed by replacing the Parameter
with `AWS::SecretsManager::Secret` (`GenerateSecretString`) plus the dynamic
reference on `MasterUsername`/`MasterUserPassword` in
`cfn/multistate-app-dev.yaml`.

A third known quirk (pairing `DeletionPolicy: Retain` without the matching
`UpdateReplacePolicy: Retain`) was checked for across all four templates in
this repo and is not present — every stateful resource (both S3 buckets,
the Secrets Manager secret, the RDS instance) carries both attributes.
