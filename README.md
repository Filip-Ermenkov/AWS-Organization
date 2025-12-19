# AWS Organization Setup

**Based on:** org-formation + AWS Identity Center (SSO)\
**Owner:** Filip Ermenkov\
**Purpose:** Production-focused organization configuration: Use Org-Formation to enable scalability for my AWS Organization and use IAM Identity Center to enable Single Sign-On (SSO) for easy access.

---

## Table of contents

1. [At-a-glance summary](#at-a-glance-summary)
2. [Architecture (diagram + explanation)](#architecture-diagram--explanation)
3. [Core templates & resources](#core-templates--resources)
4. [CI/CD & authentication model](#cicd--authentication-model)
5. [Pre-deploy checks & run instructions](#pre-deploy-checks--run-instructions)
6. [Security & governance highlights](#security--governance-highlights)
7. [Troubleshooting quick commands](#troubleshooting-quick-commands)
8. [Estimated costs](#estimated-costs)
9. [Contacts & license](#contacts--license)

---

## At-a-glance summary

* **Management account (owner):**
  * `filip-ermenkov`
    * account ID `744747681339`
    * root email `ermenkov.aws@gmail.com`.
* **OUs:**
  * `OU-Production`
  * `OU-Test`.
* **Member accounts:**

  * `acc-production`
    * account ID `472387458718`
    * root email `ermenkov.aws+production@gmail.com`.
  * `acc-test`
    * account ID `176971015975`
    * root email `ermenkov.aws+test@gmail.com`.
* **Identity provider / SSO:** AWS Identity Center configured via `identity-center.yml` (explicit Instance ARN in templates).
* **IaC tooling:** `aws-organization-formation` (org-formation) for Organization bootstrap; CloudFormation for Identity Center resources.
* **CI/CD:** GitHub Actions workflows:

  * `organization-cicd.yml` — validates and applies org-formation template.
  * `identity-center-cicd.yml` — deploys the Identity Center CloudFormation template.
* **Assume roles used by CI:**

  * `arn:aws:iam::744747681339:role/GitHubActions-OrgFormation` (organization updates)
  * `arn:aws:iam::744747681339:role/GitHubActions-IdentityCenter` (Identity Center deploys)
* **Region used in workflows:** `eu-north-1` (ensure roles exist in management account for this region where applicable).

---

## Architecture (diagram + explanation)

![Architecture Diagram](https://github.com/user-attachments/assets/0e4cc64d-0270-4abc-8898-94f3f38aecd4)

**Flow:**

1. Org-formation (`organization.yml`) is executed from the management account and creates the Organization root, OUs and member accounts.
2. Identity Center (SSO) resources (permission sets, assignments) are deployed via CloudFormation (`identity-center.yml`) and target member accounts by Account ID.
3. GitHub Actions leverages OIDC and short-lived credentials to assume named roles in the management account to execute org-formation commands and CloudFormation deploys.

---

## Core templates & resources

### `infrastructure/organization/organization.yml`

* Bootstraps the Organization via `OC::ORG::*` resources (org-formation format).
* Declares the ManagementAccount, OrganizationRoot, OUs, and Account resources with explicit `AccountId` and `RootEmail`.
* When updating: `npx org-formation validate` followed by `npx org-formation update` (see CI steps).

### `infrastructure/identity-center/identity-center.yml`

* CloudFormation template that creates PermissionSet(s) and Assignment(s) for Identity Center (SSO).
* Uses a fixed `InstanceArn` (ensure the ARN in the template matches the live Identity Center instance in the management account).
* Assignments reference `TargetId` as the target AWS account IDs (e.g., `472387458718`, `176971015975`).

### CI Workflows

* `organization-cicd.yml` — checks out repository, configures AWS credentials via `aws-actions/configure-aws-credentials@v4`, installs `aws-organization-formation`, validates and runs `org-formation update` on the organization template.
* `identity-center-cicd.yml` — checks out, configures AWS creds, runs `aws cloudformation deploy` with `CAPABILITY_NAMED_IAM` and `--no-fail-on-empty-changeset`.

---

## CI/CD & authentication model

* **OIDC with GitHub Actions:** Workflows are configured with `permissions: id-token: write` and `aws-actions/configure-aws-credentials` to assume IAM roles in the management account. This eliminates long-lived keys.
* **Role separation:** One role for organization updates; one role for identity center deployments. Each role should follow least-privilege.
* **Path filtering:** Workflows are path-filtered to limit runs to when relevant files change.

---

## Pre-deploy checks & run instructions

### Local validation

```bash
# validate YAML and org-formation template
npm install aws-organization-formation
npx org-formation validate infrastructure/organization/organization.yml
```

### CI

* Push changes to `main` on paths matching the workflows; workflows will run automatically and assume configured roles.

### Manual CloudFormation deploy (Identity Center)

```bash
aws cloudformation deploy \
  --template-file infrastructure/identity-center/identity-center.yml \
  --stack-name identity-center \
  --capabilities CAPABILITY_NAMED_IAM \
  --no-fail-on-empty-changeset
```

**Important preconditions before deploy:**

* Ensure the GitHub Action roles (`GitHubActions-OrgFormation`, `GitHubActions-IdentityCenter`) exist with the correct trust policy for GitHub OIDC.
* Ensure Identity Center `InstanceArn` in `identity-center.yml` matches the account where Identity Center is enabled.
* Ensure the account IDs in `organization.yml` are correct and you control the root emails / have access to accept invites (if creating new accounts).

---

## Security & governance highlights

* **Least privilege:** CI roles are scoped narrowly to just the actions they need (org-formation actions, CloudFormation deploys).
* **Authentication:** Usage of GitHub Actions OIDC.
* **Account roots:** Protecting management account root user with MFA.

---

## Troubleshooting quick commands

**Org-formation validation locally**

```bash
npx org-formation validate infrastructure/organization/organization.yml
```

**View CloudFormation stack events (Identity Center)**

```bash
aws cloudformation describe-stack-events --stack-name identity-center --max-items 50
```

**Check recent Organization changes via AWS CLI**

```bash
aws organizations list-roots
aws organizations list-accounts-for-parent --parent-id <root-id> --query 'Accounts[*].[Id,Name,Status]'
```

**Inspect IAM role trust policy**

```bash
aws iam get-role --role-name GitHubActions-OrgFormation --query 'Role.AssumeRolePolicyDocument' --output json
```

---

## Estimated costs

* The organization bootstrap itself is effectively free — costs arise from resources you create after account creation (CloudTrail centralized S3, Config aggregator, Control Tower, AWS Managed Services).
* Identity Center resources have minimal direct cost; permission sets and assignments are free; costs come from services accessed by assigned roles.

---

## Contacts & license

**Owner:** Filip Ermenkov — [f.ermenkov@gmail.com](mailto:f.ermenkov@gmail.com)\
**License:** This project is licensed under the [MIT License](LICENSE)