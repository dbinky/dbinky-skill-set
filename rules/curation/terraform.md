# Terraform Curation Rules

## Idiomatic Patterns

What senior Terraform engineers write:

- **Modules for reuse**: Extract repeated resource groups into modules with clear `variable` inputs and `output` blocks. Module contracts should be narrow — pass only what the module needs, expose only what callers consume.
- **Variable validation blocks**: Use `validation` blocks inside `variable` to enforce allowed values, ranges, or naming conventions at plan time rather than relying on provider errors at apply time.
- **`locals` for computed values**: Derive repeated or complex expressions once in a `locals` block and reference the local name everywhere else. Never duplicate the same expression in two places.
- **`for_each` over `count`**: Use `for_each` with a map or set when creating multiple resources. `count`-indexed resources are fragile — removing a middle item renumbers all subsequent resources and triggers unwanted destroys.
- **Explicit lifecycle rules**: Add `lifecycle { prevent_destroy = true }` to stateful resources (databases, S3 buckets, KMS keys) so Terraform refuses destructive applies without a manual override.
- **Data sources for lookups**: Use `data` sources to fetch AMI IDs, VPC IDs, availability zones, and other environment-specific values instead of hardcoding them. This keeps configurations portable across accounts and regions.
- **Consistent naming**: Use `snake_case` for all resource labels, variable names, output names, and locals. Include the resource type in the label where it aids clarity (e.g., `web_security_group`, not just `web`).
- **Minimal outputs**: Export only values that other modules or root configurations actually consume. Outputting everything creates implicit coupling and leaks internal details.
- **`terraform fmt`**: Run `terraform fmt -recursive` before committing. Consistent indentation and alignment are enforced by the tool, not by convention.
- **Remote state with locking**: Store state in a remote backend (S3 + DynamoDB, Terraform Cloud, GCS) with locking enabled. Never use local state in shared environments.

## Vibe-Code Anti-Patterns

Common patterns LLM-assisted coding produces in Terraform:

- **Hardcoded environment values**: Embedding specific region names, account IDs, AMI IDs, or CIDR blocks directly in resource blocks. These values drift, break in other accounts, and cannot be overridden without editing source files.
- **`count` conditionals**: `count = var.enabled ? 1 : 0` creates index-addressed resources. Enabling/disabling is fine, but any list-based `count` leads to cascading destroy-and-recreate when the source list changes.
- **Monolithic root modules**: A single `main.tf` exceeding 500 lines that mixes networking, compute, IAM, and data tiers. Difficult to review, impossible to reuse, and error-prone to modify.
- **Unused variables**: Variables declared but never referenced by any resource, local, or output. Terraform does not warn about these; they accumulate and confuse readers about what is actually configurable.
- **Redundant `depends_on`**: Explicit `depends_on` between resources where Terraform already infers the dependency through attribute references. This adds noise and can hide real ordering problems.
- **Unnecessary string interpolation**: Writing `"${var.name}"` when the value is a simple variable reference. The correct form is just `var.name`; interpolation syntax is only needed when combining with other strings.
- **Provider blocks in child modules**: Declaring `provider` blocks inside reusable modules forces a specific provider configuration on all callers. Pass providers in via the calling module instead.
- **Overly permissive IAM**: IAM policies with `"Resource": "*"` on destructive or sensitive actions. Even in development environments this pattern migrates unchanged to production.
- **Missing tags**: Resources without at minimum `Name`, environment, and team/owner tags. Missing tags make cost attribution, incident response, and automated governance impossible.

## Simplification Strategies

### 1. Replace hardcoded AMI with a data source lookup

Before — AMI ID baked into the resource:

```hcl
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_type

  tags = {
    Name = "app-server"
  }
}
```

After — data source fetches the current AMI by owner and filter:

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type

  tags = {
    Name = "app-server"
  }
}
```

### 2. Replace `count` conditional with a `for_each` set

Before — `count` used to conditionally create a resource, fragile when combined with lists:

```hcl
variable "enable_bastion" {
  type    = bool
  default = false
}

resource "aws_instance" "bastion" {
  count         = var.enable_bastion ? 1 : 0
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  subnet_id     = var.public_subnet_ids[count.index]

  tags = {
    Name = "bastion-${count.index}"
  }
}
```

After — `for_each` on an explicit set eliminates index fragility:

```hcl
variable "bastion_subnet_ids" {
  type    = set(string)
  default = []
  description = "Pass one or more subnet IDs to enable bastion hosts; leave empty to disable."
}

resource "aws_instance" "bastion" {
  for_each      = var.bastion_subnet_ids
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  subnet_id     = each.value

  tags = {
    Name = "bastion-${each.key}"
  }
}
```

### 3. Remove unnecessary string interpolation

Before — interpolation wrapping a lone variable reference:

```hcl
resource "aws_s3_bucket" "artifacts" {
  bucket = "${var.project_name}-${var.environment}-artifacts"
  acl    = "${var.bucket_acl}"

  tags = {
    Project     = "${var.project_name}"
    Environment = "${var.environment}"
  }
}
```

After — interpolation retained only where string concatenation is required:

```hcl
resource "aws_s3_bucket" "artifacts" {
  bucket = "${var.project_name}-${var.environment}-artifacts"
  acl    = var.bucket_acl

  tags = {
    Project     = var.project_name
    Environment = var.environment
  }
}
```

## Dead Code Signals

Terraform-specific indicators that configuration is no longer exercised or referenced:

- **Unused variables**: Variables declared in `variables.tf` but not referenced by any resource, local, data source, or output. `terraform validate` does not flag these; they accumulate across refactors.
- **Unused outputs**: Outputs declared but never read by a parent module via `module.<name>.<output>` or by an external consumer via remote state. Safe to delete once the consumer is confirmed gone.
- **Unused data sources**: `data` blocks that are declared and fetched but whose attributes (`data.<type>.<name>.<attr>`) are not referenced anywhere in the module. Each unused data source adds a real API call at plan time.
- **Commented-out resources**: Blocks wrapped in `#` or `/* */` instead of being removed or placed behind a variable. These create confusion about intent and often represent partially-completed migrations.
- **Unused locals**: Entries in a `locals` block whose names never appear in resource arguments, other locals, or outputs. Frequently left behind after expressions are inlined or modules are restructured.
- **Unused modules**: `module` blocks whose outputs are never consumed by the calling configuration. The module is still applied and its resources created, incurring real infrastructure cost with no observable benefit.
- **Orphaned `.tfvars` files**: Variable definition files (`*.tfvars`, `*.tfvars.json`) in the repository that are no longer passed to any `terraform apply` invocation and whose variables no longer exist in `variables.tf`. These mislead readers about the active configuration.
