# Terraform AWS IAM Module

A Terraform module for managing [AWS Identity and Access Management (IAM)](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html) building blocks. Every feature is behind its own toggle, so a single module invocation can manage the account password policy, an IAM Access Analyzer, a SAML or OIDC identity provider, a user with an access key, a customer managed policy, a role with an instance profile, or a group with members - each rendered from JSON/XML templates kept in your own repository.

## Features

- **Account Password Policy**: Optionally manages the account-wide password policy, including complexity, minimum length, reuse prevention and maximum age.
- **IAM Access Analyzer**: Optionally creates an `ACCOUNT` or `ORGANIZATION` scoped analyzer, enabling the Access Analyzer service principal on the organization when organization scope is used.
- **SAML Identity Provider**: Registers a SAML provider from a local XML metadata document resolved as `<saml_path>/<saml_name>.xml`.
- **OIDC Identity Provider**: Registers an OpenID Connect provider from a URL, audience list and certificate thumbprint list.
- **Users & Access Keys**: Creates an IAM user at a given path and, independently, a programmatic access key for any user name.
- **Templated Policies**: Renders a customer managed policy from `<policy_path>/<policy_name>.json` with `name` and `account` template variables injected.
- **Roles & Instance Profiles**: Renders a trust policy from `<role_path>/<role_name>.json`, attaches any number of managed policy ARNs, sets the maximum session duration and creates a matching instance profile.
- **Groups & Membership**: Creates an IAM group, sets its exact membership list and attaches a managed policy ARN to it.

---

## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | >= 1.0 |
| <a name="requirement_aws"></a> [aws](#requirement\_aws) | ~> 5.0 |

## Providers

| Name | Version |
|------|---------|
| <a name="provider_aws"></a> [aws](#provider\_aws) | ~> 5.0 |

---

## Usage Examples

### Basic Example

Manage only the account password policy. All other features stay disabled:

```hcl
module "iam_password_policy" {
  source = "git::https://github.com/virsas/terraform-aws-iam.git?ref=v1.1"

  accountID = "123456789012"

  password_configuration_enabled          = true
  password_configuration_minimum_length   = 20
  password_configuration_max_age          = 60
  password_configuration_reuse_prevention = 12
}
```

### User, Access Key and Group Example

Create a user, issue an access key for it and place it in a group that carries a managed policy. Because the access key is created by user name rather than by reference, keep the user creation in a separate module call or apply so that the user exists first:

```hcl
module "iam_user" {
  source = "git::https://github.com/virsas/terraform-aws-iam.git?ref=v1.1"

  accountID = "123456789012"

  user_create        = true
  user_name          = "vss-deployer"
  user_path          = "/service/"
  user_force_destroy = true
}

module "iam_user_key" {
  source = "git::https://github.com/virsas/terraform-aws-iam.git?ref=v1.1"

  accountID = "123456789012"

  access_key_create    = true
  access_key_user_name = module.iam_user.user_name
}

module "iam_group" {
  source = "git::https://github.com/virsas/terraform-aws-iam.git?ref=v1.1"

  accountID = "123456789012"

  group_create = true
  group_name   = "vss-deployers"
  group_path   = "/service/"
  group_users  = [module.iam_user.user_name]
  group_policy = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}
```

### Policy and Role Example

Render a customer managed policy and a role trust policy from local JSON templates, attach managed policies to the role and extend the session to the 12 hour maximum.

Given the following files in your repository:
- `./json/iam/policy/vss-artifact-access.json` - the permission policy document
- `./json/iam/role/vss-ec2-role.json` - the assume role (trust) policy document

Both templates receive the `name` and `account` variables, so they can reference `${name}` and `${account}`:

```hcl
module "iam_policy" {
  source = "git::https://github.com/virsas/terraform-aws-iam.git?ref=v1.1"

  accountID = "123456789012"

  policy_create = true
  policy_name   = "vss-artifact-access"
  policy_path   = "./json/iam/policy"
}

module "iam_role" {
  source = "git::https://github.com/virsas/terraform-aws-iam.git?ref=v1.1"

  accountID = "123456789012"

  role_create           = true
  role_name             = "vss-ec2-role"
  role_path             = "./json/iam/role"
  role_session_duration = 43200

  role_policies = [
    "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore",
    module.iam_policy.policy_arn
  ]
}
```

### Advanced Example (Identity Providers & Access Analyzer)

Register a SAML provider from a local metadata document, register an OIDC provider for GitHub Actions and enable an organization wide Access Analyzer.

The SAML metadata document is expected at `./xml/iam/VssOkta.xml`:

```hcl
module "iam_identity" {
  source = "git::https://github.com/virsas/terraform-aws-iam.git?ref=v1.1"

  accountID = "123456789012"

  saml_enabled = true
  saml_name    = "VssOkta"
  saml_path    = "./xml/iam"

  oidc_enabled         = true
  oidc_url             = "https://token.actions.githubusercontent.com"
  oidc_client_id_list  = ["sts.amazonaws.com"]
  oidc_thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]

  analyzer_enabled = true
  analyzer_type    = "ORGANIZATION"
  analyzer_name    = "VssOrgAccessAnalyzer"
}
```

> [!NOTE]
> Setting `analyzer_type = "ORGANIZATION"` also makes the module manage an `aws_organizations_organization` resource that enables the `access-analyzer.amazonaws.com` service principal. Only use it from the organization management account, and be aware that Terraform will then consider the organization itself part of this module's state.

---

## Resources

| Name | Type |
|------|------|
| [aws_accessanalyzer_analyzer.vss](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/accessanalyzer_analyzer) | resource |
| [aws_accessanalyzer_analyzer.vss_org](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/accessanalyzer_analyzer) | resource |
| [aws_iam_access_key.vss](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_access_key) | resource |
| [aws_iam_account_password_policy.vss](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_account_password_policy) | resource |
| [aws_iam_group.vss](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_group) | resource |
| [aws_iam_group_membership.vss](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_group_membership) | resource |
| [aws_iam_group_policy_attachment.vss](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_group_policy_attachment) | resource |
| [aws_iam_instance_profile.vss](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_instance_profile) | resource |
| [aws_iam_openid_connect_provider.vss](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_openid_connect_provider) | resource |
| [aws_iam_policy.vss](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_policy) | resource |
| [aws_iam_role.vss](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role) | resource |
| [aws_iam_role_policy_attachment.vss](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy_attachment) | resource |
| [aws_iam_saml_provider.vss](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_saml_provider) | resource |
| [aws_iam_user.vss](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_user) | resource |
| [aws_organizations_organization.vss_org](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/organizations_organization) | resource |

---

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_accountID"></a> [accountID](#input\_accountID) | The AWS account ID, injected into the policy and role JSON templates as the `account` variable. | `string` | `""` | no |
| <a name="input_password_configuration_enabled"></a> [password\_configuration\_enabled](#input\_password\_configuration\_enabled) | If set to `true`, the account password policy is managed based on the `password_configuration_*` values below. | `bool` | `false` | no |
| <a name="input_password_configuration_allow_users_to_change_password"></a> [password\_configuration\_allow\_users\_to\_change\_password](#input\_password\_configuration\_allow\_users\_to\_change\_password) | Whether IAM users are allowed to change their own passwords. | `bool` | `true` | no |
| <a name="input_password_configuration_require_numbers"></a> [password\_configuration\_require\_numbers](#input\_password\_configuration\_require\_numbers) | Whether passwords must contain at least one digit. | `bool` | `true` | no |
| <a name="input_password_configuration_require_symbols"></a> [password\_configuration\_require\_symbols](#input\_password\_configuration\_require\_symbols) | Whether passwords must contain at least one symbol. | `bool` | `true` | no |
| <a name="input_password_configuration_require_lowercase_characters"></a> [password\_configuration\_require\_lowercase\_characters](#input\_password\_configuration\_require\_lowercase\_characters) | Whether passwords must contain at least one lowercase character. | `bool` | `true` | no |
| <a name="input_password_configuration_require_uppercase_characters"></a> [password\_configuration\_require\_uppercase\_characters](#input\_password\_configuration\_require\_uppercase\_characters) | Whether passwords must contain at least one uppercase character. | `bool` | `true` | no |
| <a name="input_password_configuration_minimum_length"></a> [password\_configuration\_minimum\_length](#input\_password\_configuration\_minimum\_length) | Minimum length of an IAM user password. AWS accepts values from `6` to `128`. | `number` | `16` | no |
| <a name="input_password_configuration_reuse_prevention"></a> [password\_configuration\_reuse\_prevention](#input\_password\_configuration\_reuse\_prevention) | How many previous passwords a user must cycle through before an old one can be reused. AWS accepts values from `1` to `24`. | `number` | `24` | no |
| <a name="input_password_configuration_max_age"></a> [password\_configuration\_max\_age](#input\_password\_configuration\_max\_age) | How many days a password can be used before the user is required to change it. AWS accepts values from `1` to `1095`. | `number` | `90` | no |
| <a name="input_analyzer_enabled"></a> [analyzer\_enabled](#input\_analyzer\_enabled) | Enable IAM Access Analyzer. | `bool` | `false` | no |
| <a name="input_analyzer_type"></a> [analyzer\_type](#input\_analyzer\_type) | Scope of the analyzer. Allowed values are `ACCOUNT` and `ORGANIZATION`. | `string` | `"ACCOUNT"` | no |
| <a name="input_analyzer_name"></a> [analyzer\_name](#input\_analyzer\_name) | Name of the analyzer. | `string` | `"AccessAnalyzer"` | no |
| <a name="input_saml_enabled"></a> [saml\_enabled](#input\_saml\_enabled) | Enable IAM SAML provider configuration. | `bool` | `false` | no |
| <a name="input_saml_name"></a> [saml\_name](#input\_saml\_name) | Name of the SAML provider, also used for the metadata file name, so the module reads `<saml_path>/<saml_name>.xml`. | `string` | `"SamlProvider"` | no |
| <a name="input_saml_path"></a> [saml\_path](#input\_saml\_path) | Directory path holding the SAML metadata XML document. | `string` | `"./xml/iam"` | no |
| <a name="input_user_create"></a> [user\_create](#input\_user\_create) | Enable IAM user creation. | `bool` | `false` | no |
| <a name="input_user_name"></a> [user\_name](#input\_user\_name) | Name of the IAM user. | `string` | `"user"` | no |
| <a name="input_user_path"></a> [user\_path](#input\_user\_path) | Path in which to create the user. | `string` | `"/"` | no |
| <a name="input_user_force_destroy"></a> [user\_force\_destroy](#input\_user\_force\_destroy) | Whether to destroy the user even if they created access keys or login profiles outside of Terraform. | `bool` | `true` | no |
| <a name="input_access_key_create"></a> [access\_key\_create](#input\_access\_key\_create) | Enable access key creation. | `bool` | `false` | no |
| <a name="input_access_key_user_name"></a> [access\_key\_user\_name](#input\_access\_key\_user\_name) | Name of the existing IAM user for whom to create the access key. | `string` | `"user"` | no |
| <a name="input_policy_create"></a> [policy\_create](#input\_policy\_create) | Enable customer managed policy creation. The resulting policy can then be attached to a group or role. | `bool` | `false` | no |
| <a name="input_policy_name"></a> [policy\_name](#input\_policy\_name) | Name of the policy, also used for the document file name, so the module reads `<policy_path>/<policy_name>.json`. | `string` | `"policy"` | no |
| <a name="input_policy_environment"></a> [policy\_environment](#input\_policy\_environment) | Intended to select an environment specific document (`<policy_name>-<policy_environment>.json`) while keeping the policy named after `policy_name`. Currently unused by the module code. | `string` | `""` | no |
| <a name="input_policy_path"></a> [policy\_path](#input\_policy\_path) | Directory path holding the policy JSON documents. | `string` | `"./json/iam/policy"` | no |
| <a name="input_role_create"></a> [role\_create](#input\_role\_create) | Enable role creation. Also creates a matching instance profile and the managed policy attachments. | `bool` | `false` | no |
| <a name="input_role_name"></a> [role\_name](#input\_role\_name) | Name of the role, also used for the instance profile name and the trust policy file name, so the module reads `<role_path>/<role_name>.json`. | `string` | `"role"` | no |
| <a name="input_role_environment"></a> [role\_environment](#input\_role\_environment) | Intended to select an environment specific document (`<role_name>-<role_environment>.json`) while keeping the role named after `role_name`. Currently unused by the module code. | `string` | `""` | no |
| <a name="input_role_path"></a> [role\_path](#input\_role\_path) | Directory path holding the role trust policy JSON documents. | `string` | `"./json/iam/role"` | no |
| <a name="input_role_policies"></a> [role\_policies](#input\_role\_policies) | List of managed policy ARNs to attach to the role. Can be AWS managed or customer managed. | `list(string)` | `[]` | no |
| <a name="input_role_session_duration"></a> [role\_session\_duration](#input\_role\_session\_duration) | Maximum session duration in seconds for the role. Accepted values are whole numbers from `3600` (1 hour) to `43200` (12 hours). | `number` | `3600` | no |
| <a name="input_group_create"></a> [group\_create](#input\_group\_create) | Enable group creation. Also creates the group membership and the group policy attachment. | `bool` | `false` | no |
| <a name="input_group_name"></a> [group\_name](#input\_group\_name) | Name of the IAM group. The membership resource is named `<group_name>-membership`. | `string` | `"group"` | no |
| <a name="input_group_path"></a> [group\_path](#input\_group\_path) | Path in which to create the group. | `string` | `"/"` | no |
| <a name="input_group_users"></a> [group\_users](#input\_group\_users) | List of user names that make up the exact membership of this group. | `list(string)` | `[]` | no |
| <a name="input_group_policy"></a> [group\_policy](#input\_group\_policy) | ARN of a managed policy to attach to this group. Required in practice whenever `group_create` is `true`. | `string` | `""` | no |
| <a name="input_oidc_enabled"></a> [oidc\_enabled](#input\_oidc\_enabled) | Enable OIDC identity provider creation. | `bool` | `false` | no |
| <a name="input_oidc_url"></a> [oidc\_url](#input\_oidc\_url) | URL of the OIDC provider, including the `https://` scheme. | `string` | `""` | no |
| <a name="input_oidc_client_id_list"></a> [oidc\_client\_id\_list](#input\_oidc\_client\_id\_list) | List of client IDs (audiences) associated with the OIDC provider. | `list(string)` | `[]` | no |
| <a name="input_oidc_thumbprint_list"></a> [oidc\_thumbprint\_list](#input\_oidc\_thumbprint\_list) | List of server certificate thumbprints for the OIDC provider's certificates. | `list(string)` | `[]` | no |

---

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_saml_arn"></a> [saml\_arn](#output\_saml\_arn) | ARN of the SAML provider, or an empty string when SAML is disabled. |
| <a name="output_saml_valid_until"></a> [saml\_valid\_until](#output\_saml\_valid\_until) | Expiration date and time of the SAML metadata document. |
| <a name="output_user_arn"></a> [user\_arn](#output\_user\_arn) | ARN of the IAM user, or an empty string when user creation is disabled. |
| <a name="output_user_name"></a> [user\_name](#output\_user\_name) | Name of the IAM user. |
| <a name="output_user_unique_id"></a> [user\_unique\_id](#output\_user\_unique\_id) | Unique ID assigned by AWS to the IAM user. |
| <a name="output_access_key_create_date"></a> [access\_key\_create\_date](#output\_access\_key\_create\_date) | Date and time the access key was created. |
| <a name="output_access_key_id"></a> [access\_key\_id](#output\_access\_key\_id) | ID of the access key. |
| <a name="output_access_key_secret"></a> [access\_key\_secret](#output\_access\_key\_secret) | Secret access key. Treat the Terraform state as sensitive. |
| <a name="output_access_key_encrypted_secret"></a> [access\_key\_encrypted\_secret](#output\_access\_key\_encrypted\_secret) | PGP encrypted secret access key, populated only when a PGP key was supplied to the access key resource. |
| <a name="output_access_key_key_fingerprint"></a> [access\_key\_key\_fingerprint](#output\_access\_key\_key\_fingerprint) | Fingerprint of the PGP key used to encrypt the secret. |
| <a name="output_access_key_ses_smtp_password_v4"></a> [access\_key\_ses\_smtp\_password\_v4](#output\_access\_key\_ses\_smtp\_password\_v4) | Secret access key converted into an SES SMTP password using the Signature Version 4 algorithm. |
| <a name="output_access_key_encrypted_ses_smtp_password_v4"></a> [access\_key\_encrypted\_ses\_smtp\_password\_v4](#output\_access\_key\_encrypted\_ses\_smtp\_password\_v4) | PGP encrypted SES SMTP password. |
| <a name="output_policy_id"></a> [policy\_id](#output\_policy\_id) | ID of the customer managed policy. |
| <a name="output_policy_arn"></a> [policy\_arn](#output\_policy\_arn) | ARN of the customer managed policy. |
| <a name="output_policy_name"></a> [policy\_name](#output\_policy\_name) | Name of the customer managed policy. |
| <a name="output_policy_policy"></a> [policy\_policy](#output\_policy\_policy) | Rendered JSON document of the customer managed policy. |
| <a name="output_policy_policy_id"></a> [policy\_policy\_id](#output\_policy\_policy\_id) | Policy ID assigned by AWS to the customer managed policy. |
| <a name="output_role_id"></a> [role\_id](#output\_role\_id) | ID of the IAM role. |
| <a name="output_role_arn"></a> [role\_arn](#output\_role\_arn) | ARN of the IAM role. |
| <a name="output_role_name"></a> [role\_name](#output\_role\_name) | Name of the IAM role. |
| <a name="output_role_unique_id"></a> [role\_unique\_id](#output\_role\_unique\_id) | Unique ID assigned by AWS to the IAM role. |
| <a name="output_profile_id"></a> [profile\_id](#output\_profile\_id) | ID of the instance profile created for the role. |
| <a name="output_profile_arn"></a> [profile\_arn](#output\_profile\_arn) | ARN of the instance profile created for the role. |
| <a name="output_group_id"></a> [group\_id](#output\_group\_id) | ID of the IAM group. |
| <a name="output_group_arn"></a> [group\_arn](#output\_group\_arn) | ARN of the IAM group. |
| <a name="output_group_name"></a> [group\_name](#output\_group\_name) | Name of the IAM group. |
| <a name="output_group_unique_id"></a> [group\_unique\_id](#output\_group\_unique\_id) | Unique ID assigned by AWS to the IAM group. |
| <a name="output_group_membership_name"></a> [group\_membership\_name](#output\_group\_membership\_name) | Name of the group membership resource. |
| <a name="output_group_membership_users"></a> [group\_membership\_users](#output\_group\_membership\_users) | List of user names attached to the group. |
| <a name="output_group_membership_group"></a> [group\_membership\_group](#output\_group\_membership\_group) | Name of the group the membership applies to. |
| <a name="output_oidc_provider_arn"></a> [oidc\_provider\_arn](#output\_oidc\_provider\_arn) | ARN of the OIDC identity provider. |
| <a name="output_oidc_provider_url"></a> [oidc\_provider\_url](#output\_oidc\_provider\_url) | URL of the OIDC identity provider. |
| <a name="output_oidc_provider_client_id_list"></a> [oidc\_provider\_client\_id\_list](#output\_oidc\_provider\_client\_id\_list) | List of client IDs (audiences) registered with the OIDC identity provider. |
| <a name="output_oidc_provider_thumbprint_list"></a> [oidc\_provider\_thumbprint\_list](#output\_oidc\_provider\_thumbprint\_list) | List of certificate thumbprints registered with the OIDC identity provider. |

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
