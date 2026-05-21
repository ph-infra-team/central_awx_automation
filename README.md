# central_awx_automation

Trusted central AWX/AAP configuration project.

This repository is owned by the platform team and is used by AWX job templates
to create, update, and clean up AWX resources requested by application
automation repositories.

Application teams do not edit this repository. They declare their requested AWX
resources in their own repository, usually in `awx/team_vars.yml`, and consume
the shared GitLab pipeline `platform-pipelines/ansible/awx-app.yml`.

## AWX Resources To Create

Create these once in the `central_awx` organization.

These are platform-owned bootstrap resources. Application engineers should not
create or edit these by hand. Their app repos pass approved variables into these
templates through the shared GitLab pipeline.

### Project

```text
Name: central-awx-configure
Organization: central_awx
SCM URL: http://gitlab.midhtech.local/infra_team/central_awx_automation.git
SCM Branch: main
```

Use a GitLab source control credential if the repository is private.

### Job Template: Configure

```text
Name: central-awx-configure-resources
Organization: central_awx
Project: central-awx-configure
Playbook: playbooks/controller_config.yml
Inventory: localhost/controller inventory
Execution Environment: AWX EE or a platform AWX automation EE
Ask Variables On Launch: enabled
Concurrent Jobs: enabled
```

Attach a trusted AWX controller credential that can create/update AWX resources.

### Job Template: Cleanup

```text
Name: central-awx-cleanup-resources
Organization: central_awx
Project: central-awx-configure
Playbook: playbooks/controller_cleanup.yml
Inventory: localhost/controller inventory
Execution Environment: AWX EE or a platform AWX automation EE
Ask Variables On Launch: enabled
Concurrent Jobs: enabled
```

## Controller Credential

The central configure and cleanup templates need a trusted controller credential.
The job output must not show `credentials: []`; if it does, the template cannot
authenticate back to AWX to create resources.

Preferred injection method: environment variables.

```text
CONTROLLER_HOST
CONTROLLER_USERNAME
CONTROLLER_PASSWORD
CONTROLLER_VERIFY_SSL
```

Supported alternate injection method: extra variables.

```text
controller_host
controller_username
controller_password
controller_verify_ssl
```

Create a custom AWX credential type for this if needed, then attach that
credential to both central job templates:

```text
central-awx-configure-resources
central-awx-cleanup-resources
```

## AWS AssumeRole Credential Type

Create this custom credential type once in AWX. It is used by app job templates
that connect to AWS through a central base AWS credential plus a team role ARN.

Credential type name:

```text
AWS Assume Role Profile
```

Input configuration:

```yaml
fields:
  - id: profile_name
    type: string
    label: AWS Profile Name
  - id: role_arn
    type: string
    label: Role ARN
  - id: region
    type: string
    label: AWS Region
required:
  - profile_name
  - role_arn
  - region
```

Injector configuration:

```yaml
env:
  AWS_CONFIG_FILE: "{{ tower.filename.aws_config }}"
file:
  template.aws_config: |
    [profile {{ profile_name }}]
    role_arn = {{ role_arn }}
    credential_source = Environment
    region = {{ region }}
```

The app job template must attach both credentials:

```text
central base AWS credential
team assume-role profile credential
```

The base AWS credential supplies `AWS_ACCESS_KEY_ID` and
`AWS_SECRET_ACCESS_KEY`. The assume-role profile credential supplies
`AWS_CONFIG_FILE` and tells boto3 which team role to assume.

## AWS Bootstrap

Create one central AWS IAM user for AWX, then let that user assume team roles.
This keeps long-term team AWS keys out of GitLab variables and out of Ansible
Vault files.

### IAM User

Create this once in AWS:

```text
User name: awx-automation
Console access: disabled
Access key: enabled
```

When AWS asks for the access key use case, choose:

```text
Application running outside AWS
```

AWX is outside AWS in this environment, so this is the correct category for the
base credential. Store the generated access key only in AWX.

Attach a policy that allows the user to assume approved automation roles:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": [
        "arn:aws:iam::288821543174:role/awx-ssm-automation-role"
      ]
    }
  ]
}
```

For multiple AWS accounts, add only the approved role ARNs. Do not grant
wildcard account access unless the platform team explicitly accepts that risk.

### Team Role

Create one role per AWS account or team boundary:

```text
Role name: awx-ssm-automation-role
Trusted principal: arn:aws:iam::288821543174:user/awx-automation
```

Example trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::288821543174:user/awx-automation"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Attach the operational permissions to the role, not to the IAM user:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:StartSession",
        "ssm:TerminateSession",
        "ssm:DescribeSessions",
        "ssm:GetConnectionStatus",
        "ssm:SendCommand",
        "ssm:GetCommandInvocation",
        "ec2:DescribeInstances"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::tfstate-midhtech-platform"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::tfstate-midhtech-platform/*"
    }
  ]
}
```

`tfstate-midhtech-platform` is acceptable for initial testing if approved by the
platform owner. The cleaner enterprise target is a dedicated encrypted SSM
transfer bucket with lifecycle cleanup.

## AWX Manual Bootstrap

Create these manually once, then manage app resources through GitLab pipelines.

### Controller Credential

Create a custom AWX credential type that injects the controller connection into
central jobs:

```text
Name: AWX Controller Admin
```

Suggested fields:

```yaml
fields:
  - id: controller_host
    type: string
    label: Controller Host
  - id: controller_username
    type: string
    label: Controller Username
  - id: controller_password
    type: string
    label: Controller Password
    secret: true
  - id: controller_verify_ssl
    type: boolean
    label: Verify SSL
required:
  - controller_host
  - controller_username
  - controller_password
```

Suggested injector:

```yaml
env:
  CONTROLLER_HOST: "{{ controller_host }}"
  CONTROLLER_USERNAME: "{{ controller_username }}"
  CONTROLLER_PASSWORD: "{{ controller_password }}"
  CONTROLLER_VERIFY_SSL: "{{ controller_verify_ssl | default(false) }}"
```

Create the credential:

```text
Name: central-awx-controller-admin
Credential Type: AWX Controller Admin
Organization: central_awx
CONTROLLER_HOST: http://awx.midhtech.local
CONTROLLER_USERNAME: <awx service account>
CONTROLLER_PASSWORD: <secret>
CONTROLLER_VERIFY_SSL: false
```

Attach it to:

```text
central-awx-configure-resources
central-awx-cleanup-resources
```

### Base AWS Credential

Create one base AWS credential in AWX using the `awx-automation` IAM access key:

```text
Name: central-aws-assume-role-base
Credential Type: Amazon Web Services
Organization: central_awx or shared platform organization
Access Key: <awx-automation access key>
Secret Key: <awx-automation secret key>
```

This credential is not the team permission boundary. It only proves identity and
allows STS AssumeRole into approved team roles.

### Source Control Credential

Create or reuse a GitLab source control credential for private automation repos:

```text
Name: git_root_access_token
Credential Type: Source Control
Username: <gitlab user or token user>
Password: <gitlab token>
```

Application projects reference this by name through the GitLab group variable:

```text
AWX_SCM_CREDENTIAL_NAME=git_root_access_token
```

## GitLab Group Variables

Set these at the `infra_team/automation` group or another controlled parent
group so junior engineers do not manage AWX secrets in each repo:

```text
AWX_HOST=http://awx.midhtech.local
AWX_USERNAME=<awx service account>
AWX_PASSWORD=<masked secret>
AWX_SCM_CREDENTIAL_NAME=git_root_access_token
AWX_BASE_AWS_CREDENTIAL_NAME=central-aws-assume-role-base
AWX_SSM_BUCKET_NAME=tfstate-midhtech-platform
AWS_DEFAULT_REGION=us-east-1
```

Set these as protected where practical. If variables are protected, the app
repo's `main` branch must also be protected or GitLab will not expose them to
the pipeline.

Each team/app then supplies only its role ARN:

```text
AWS_ASSUME_ROLE_ARN=arn:aws:iam::288821543174:role/awx-ssm-automation-role
```

The role ARN is not a secret, but keeping it as a CI/CD variable lets the same
`awx/team_vars.yml` template work across teams and accounts.

## Collection Dependency

This project uses:

```text
http://gitlab.midhtech.local/infra_team/awx_team.config_as_code_template.git
```

For CI validation, prefer `CI_JOB_TOKEN` through Git URL rewriting.

Required GitLab setting:

```text
Project: infra_team/awx_team.config_as_code_template
Settings -> CI/CD -> Job token permissions
Allow access from:
  infra_team/central_awx_automation
```

If project job token access is not allowed, create a masked/protected GitLab
CI/CD variable in `central_awx_automation`:

```text
COLLECTION_REPOSITORY_TOKEN
```

The token must have read access to:

```text
infra_team/awx_team.config_as_code_template
```

For AWX runtime, use one of these enterprise options:

- publish the collection to an internal Automation Hub/Galaxy
- build a trusted AWX configuration EE with the collection preinstalled
- allow the AWX project credential to read the collection repository

## Consumer Flow

An app repo such as `test_awx_aws_connect` includes:

```yaml
include:
  - project: infra_team/platform-pipelines
    ref: main
    file: ansible/awx-app.yml
```

The app repo owns only:

```text
awx/team_vars.yml
playbooks/
group_vars/
```

The shared pipeline launches `central-awx-configure-resources`, syncs the app
AWX project, and can launch the app job template while streaming output back to
GitLab CI.
