# Terraform Module for AWS RDS with PostgreSQL

Create a RDS instance with PostgreSQL engine and configure it to create roles & databases.

## Usecases

### Adding a database to existing RDS instance

As an example we will add `ehr-integration` database to shared RDS instance on staging.

1. It is required to have access to one of AWS accounts as for example `KlaraProductDeveloperAccess` role. This account is used currently used for testing applications in EKS.

2. Verify that access to `terraform/password-encryption` KMS key is present. This KMS key is used to encrypt password. As alternative SRE team can help to generate the password.

3. In order to encrypt the password, execute [this](https://github.com/ModMedPD/klara-iac-scripts/blob/main/infra-scripts/encrypt_tool.sh) script i.e `encrypt_tool.sh`. The script can automatically generate a random string & encrypt it as well as perform encryption of the provided string.

4. Add the generated password to `.tfvars` file in the repository `terraform-app-network` under `stg-network/eu-central-1/rds-postgresql-env1` along with role name that needs to be created. Note that we can perform the same action under `us-east-1` directory.

   Sample:

   ```hcl
   roles = {
     casper = {
       password = "<encrypted string>"
     }
   }
   ```

   Multiple roles can be added inside the roles section.

5. Developers can also exclude the password parameter & just define the role name. This will save the hassle of performing Step 1-3. In this scenario, the terraform module will automatically generate a password and store it in AWS Secrets Manager.

   Sample:

   ```hcl
   roles = {
     casper = {}
   }
   ```

6. Ensure an entry is also made for the `databases` parameter in similar manner by specifying extensions & privileges if applicable. Refer existing entry in the tfvars.

7. Please run `terraform fmt` from the current working directory to properly format the code changes. Now the code can be committed & pushed to GitHub.

8. Upon creation of Pull Request, the changes are planned by `Atlantis` and it prints the output of terraform plan to the PR. Verify there are no unrequired changes except for the new additions.

9. Before applying the changes, please get it reviewed from Platform team in order to ensure the integrity of the system.

10. Once approval has been given, comment `atlantis apply` in the pull request to apply the planned changes.

- New database will be created inside RDS instance

- Secrets for this database will be created automatically in secret manager under `rds/instance_name/role_name/db_name` path

11. To verify that DB is created, connect to it using credentials from secret manager. Note that RDS is not accessible outside of VPC network. Therefore, it is required to have access to Client VPN(still under development) or some `pod` with `psql`.

### Tip

<details>

<summary> :bulb: How to get pod with psql installed :bulb: </summary>

You can execute this command to receive such pod:

`aws-vault exec stg-network -- kubectl run --image postgres:latest --command pg -- sleep 1000`

Then connect to pod and use `psql` + credentials to check DB works, then delete the pod.

</details>

<!-- BEGIN_TF_DOCS -->
## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | >= 1.5.0 |
| <a name="requirement_aws"></a> [aws](#requirement\_aws) | >= 4.0 |
| <a name="requirement_random"></a> [random](#requirement\_random) | ~> 3.0 |

## Providers

| Name | Version |
|------|---------|
| <a name="provider_aws"></a> [aws](#provider\_aws) | >= 4.0 |
| <a name="provider_random"></a> [random](#provider\_random) | ~> 3.0 |

## Modules

| Name | Source | Version |
|------|--------|---------|
| <a name="module_admin_credential_secret"></a> [admin\_credential\_secret](#module\_admin\_credential\_secret) | git::ssh://git@github.com/ModMedPD/terraform-aws-secret-manager | 1.1.2 |
| <a name="module_kms"></a> [kms](#module\_kms) | git::ssh://git@github.com/ModMedPD/terraform-aws-kms | 1.0.5 |
| <a name="module_postgres"></a> [postgres](#module\_postgres) | terraform-aws-modules/rds/aws | ~> 6.0 |
| <a name="module_postgres_configuration"></a> [postgres\_configuration](#module\_postgres\_configuration) | ./modules/postgresql-configuration | n/a |
| <a name="module_postgres_security_group"></a> [postgres\_security\_group](#module\_postgres\_security\_group) | terraform-aws-modules/security-group/aws | ~> 5.0 |
| <a name="module_role_credential_secret"></a> [role\_credential\_secret](#module\_role\_credential\_secret) | git::ssh://git@github.com/ModMedPD/terraform-aws-secret-manager | 1.1.2 |
| <a name="module_transfer_rds_logs_to_s3"></a> [transfer\_rds\_logs\_to\_s3](#module\_transfer\_rds\_logs\_to\_s3) | terraform-aws-modules/lambda/aws | ~> 7.0 |

## Resources

| Name | Type |
|------|------|
| [aws_default_tags.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/default_tags) | data source |
| [aws_iam_policy_document.kms](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document) | data source |
| [aws_kms_secrets.admin_credential](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/kms_secrets) | data source |
| [aws_kms_secrets.role_credentials](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/kms_secrets) | data source |
| [aws_security_groups.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/security_groups) | data source |
| [aws_subnets.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/subnets) | data source |
| [aws_vpc.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/vpc) | data source |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_configuration_enabled"></a> [configuration\_enabled](#input\_configuration\_enabled) | Bool to decide if one wants to configure PostgreSQL RDS instance with DBs,roles,permissions | `bool` | `true` | no |
| <a name="input_databases"></a> [databases](#input\_databases) | Database names created inside PostgreSQL instance | `any` | n/a | yes |
| <a name="input_enabled"></a> [enabled](#input\_enabled) | Bool to decide if RDS resources are created | `bool` | `true` | no |
| <a name="input_name"></a> [name](#input\_name) | Name of PostgreSQL instance | `string` | n/a | yes |
| <a name="input_roles"></a> [roles](#input\_roles) | Roles created inside of PostgreSQL instance | <pre>map(object({<br/>    password  = optional(string)<br/>    member_of = optional(list(string), [])<br/>  }))</pre> | `{}` | no |
| <a name="input_settings"></a> [settings](#input\_settings) | Settings of PostgreSQL instance | <pre>object({<br/>    admin_password              = optional(string)<br/>    engine_version              = optional(string, "17.2")<br/>    instance_class              = optional(string, "db.t4g.micro")<br/>    multi_az                    = optional(bool, false)<br/>    vpc_name                    = optional(string) # VPC Name ex klara/stg - if not provided name is generated from tags $product/$environment<br/>    subnet_type                 = optional(string, "database")<br/>    storage_type                = optional(string, "gp3") # Storage type gp2,gp3,io1,io2 https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Storage.html#gp3-storage<br/>    iops                        = optional(number)        # Storage size lower than 400GB can't set iops; for io storage type minimum value is 1000<br/>    throughput                  = optional(number)        # Storage size lower than 400GB can't set throughput<br/>    allocated_storage           = optional(number, 20)    # Minimal size for GP3 is 20G<br/>    backup_window               = optional(string, "04:00-05:00")<br/>    backup_retention_days       = optional(number, 7) # Snapshot retention in days<br/>    maintenance_window          = optional(string, "Mon:07:00-Mon:09:00")<br/>    source_cidrs                = list(string)               # Allowed CIDR blocks - used in security group<br/>    source_security_group_names = optional(list(string), []) # Allowed Source Security Group Names; Used as filter to query Security Group ID<br/>    log_retention_days          = optional(number, 7)        # CloudWatch log retention in days<br/>    deletion_protection         = optional(bool, true)<br/>    parameters = optional(map(object({<br/>      value        = string<br/>      apply_method = optional(string, "immediate")<br/>    })))<br/>  })</pre> | n/a | yes |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_endpoint"></a> [endpoint](#output\_endpoint) | RDS PostreSQL instance ID |
<!-- END_TF_DOCS -->
