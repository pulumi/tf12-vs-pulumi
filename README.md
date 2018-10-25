# Terraform 0.12 (HCL2) Examples in Pulumi

Terraform 0.12 features several improvements to the HCL language (dubbed HCL2), which delivers new language features.
These generally approximate existing features in the same general purpose languages that Pulumi supports -- JavaScript,
TypeScript, Python, and Go -- and so we will find that the Pulumi version is usually easier and more familiar.

# First-Class Expressions

> Prior to 0.12, expressions had to be wrapped in interpolation sequences with double quotes, such as `"${var.foo}"`.
> With 0.12, expressions are a native part of the language and can be used directly. Example: `ami = var.ami[1]`. Read
> more at https://www.hashicorp.com/blog/terraform-0-12-preview-first-class-expressions.

In Pulumi, you use real programming languages, which means you've always had first-class expressions.

## Example: First-Class Expressions

### Terraform 0.11

```hcl
variable "ami"           {}
variable "instance_type" {}
variable "vpc_security_group_ids" {
  type = "list"
}

resource "aws_instance" "example" {
  ami           = "${var.ami}"
  instance_type = "${var.instance_type}"

  vpc_security_group_ids = "${var.vpc_security_group_ids}"
}
```

### Terraform 0.12

```hcl
variable "ami"           {}
variable "instance_type" {}
variable "vpc_security_group_ids" {
  type = "list"
}

resource "aws_instance" "example" {
  ami           = var.ami
  instance_type = var.instance_type

  vpc_security_group_ids = var.vpc_security_group_ids
}
```

### Pulumi

```typescript
const config = new pulumi.Config();
const ami = config.require("ami");
const instanceType = config.require("instanceType");
const vpcSecurityGroupIds = config.require("vpcSecurityGroupIds").split(",");

const example = new aws.ec2.Instance("example", {
    ami,
    instanceType,
    vpcSecurityGroupIds,
});
```

## Example: Expressions with Lists and Maps

### Terraform 0.11

```hcl
resource "aws_instance" "example" {
  # …

  # The following works because the list structure is static
  vpc_security_group_ids = ["${var.security_group_1}", "${var.security_group_2}"]

  # The following doesn't work, because the [...] syntax isn't known to the interpolation language
  vpc_security_group_ids = "${var.security_group_id != "" ? [var.security_group_id] : []}"

  # Instead, it's necessary to use the list() function
  vpc_security_group_ids = "${var.security_group_id != "" ? list(var.security_group_id) : list()}"
}
```

### Terraform 0.12

```hcl
resource "aws_instance" "example" {
  # …

  vpc_security_group_ids = var.security_group_id != "" ? [var.security_group_id] : []
}
```

### Pulumi

```typescript
const example = new aws.ec2.Instance("example", {
    // …
    vpcSecurityGroupIds: securityGroupId ? [ securityGroupId ] : [],
});
```

# For Expressions

> A for expression is available for iterating and filtering lists and map values. This expression always can be used
> anywhere a list or map is expected. Read more at
> https://www.hashicorp.com/blog/hashicorp-terraform-0-12-preview-for-and-for-each.

In Pulumi, you get first class list comprehensions (with functions like `map`, `filter`, etc) -- even creating your own
-- and/or opt to use traditional imperative looping constructs, including `for`, `foreach`, and `while`, still
retaining all the benefits of declarative infrastructure as code.

## Example: For Expressions for List and Map Transformations

### Terraform 0.12

```hcl
variable "vpc_id" {
  description = "ID for the AWS VPC where a security group is to be created."
}

variable "subnet_numbers" {
  description = "List of 8-bit numbers of subnets of base_cidr_block that should be granted access."
  default = [1, 2, 3]
}

data "aws_vpc" "example" {
  id = var.vpc_id
}

resource "aws_security_group" "example" {
  name        = "friendly_subnets"
  description = "Allows access from friendly subnets"
  vpc_id      = var.vpc_id

  ingress {
    from_port = 0
    to_port   = 0
    protocol  = -1

    # For each number in subnet_numbers, extend the CIDR prefix of the
    # requested VPC to produce a subnet CIDR prefix.
    # For the default value of subnet_numbers above and a VPC CIDR prefix
    # of 10.1.0.0/16, this would produce:
    #   ["10.1.1.0/24", "10.1.2.0/24", "10.1.3.0/24"]
    cidr_blocks = [
      for num in var.subnet_numbers:
      cidrsubnet(data.aws_vpc.example.cidr_block, 8, num)
    ]
  }
}
```

### Pulumi

```typescript
const vpcId = config.require("vpcId");
const subnetNumbers = config.getNumbers("subnetNumbers") || [1, 2, 3];

const vpc = aws.ec2.Vpc.get(vpcId);
const example = new aws.ec2.SecurityGroup("friendly_subnets", {
    description: "Allows access from friendly subnets",
    vpcId: vpc.id,
    ingress: [
        fromPort: 0, toPort: 0, protocol: -1,
        cidrBlocks: subnetNumbers.map(num => terraform.cidrsubnet(vpc.cidrBlock, 8, num)),
    ],
});
```

## Example: For Expressions for List and Map Transformations (1/3)

### Terraform 0.12

```hcl
output "instance_private_ip_addresses" {
  # Result is a map from instance id to private IP address, such as:
  #  {"i-1234" = "192.168.1.2", "i-5678" = "192.168.1.5"}
  value = {
    for instance in aws_instance.example:
    instance.id => instance.private_ip
  }
}
```

### Pulumi

```typescript
export const instancePrivateIpAddresses = instances.map(i => [i.id, i.privateIp]).toMap();
```

## Example: For Expressions for List and Map Transformations (2/3)

### Terraform 0.12

```hcl
output "instance_public_ip_addresses" {
  value = {
    for instance in aws_instance.example:
    instance.id => instance.public
    if instance.associate_public_ip_address
  }
}
```

### Pulumi

```typescript
export const instancePublicIpAddresses =
    instances.filter(i => i.public).map(i => [i.id, i.privateIp]).toMap();
```

## Example: For Expressions for List and Map Transformations (3/3)

### Terraform 0.12

```hcl
output "instances_by_availability_zone" {
  # Result is a map from availability zone to instance ids, such as:
  #  {"us-east-1a": ["i-1234", "i-5678"]}
  value = {
    for instance in aws_instance.example:
    instance.availability_zone => instance.id...
  }
}
```

### Pulumi

```typescript
export const instancesByAvailabilityZone =
    instances.map(i => [i.availabilityZone, i.id]).toGroupedMap();
```

## Example: Resource for_each

### Terraform 0.13 (future)

```hcl
variable "subnet_numbers" {
  description = "Map from availability zone to the number that should be used for each availability zone's subnet"
  default     = {
    "eu-west-1a" = 1
    "eu-west-1b" = 2
    "eu-west-1c" = 3
  }
}

resource "aws_vpc" "example" {
  # ...
}

resource "aws_subnet" "example" {
  for_each = var.subnet_numbers

  vpc_id            = aws_vpc.example.id
  availability_zone = each.key
  cidr_block        = cidrsubnet(aws_vpc.example.cidr_block, 8, each.value)
}
```

### Pulumi (works today)

```typescript
const subnetNumbers = config.getObject("subnetNumbers") || {
    "eu-west-1a": 1,
    "eu-west-1b": 2,
    "eu-west-1c": 3,
};

const vpc = new aws.ec2.Vpc("example", {
    // ...
});

for (const az of Object.keys(subnetNumbers)) {
    new aws.ec2.Subnet(`subnet-${subnet}`, {
        vpc: vpc.id,
        availabilityZone: az,
        cidrBlock: terraform.cidrsubnet(vpc.cidrBlock, 8, subnetNumbers[az]),
    });
}
```

# Dynamic Blocks

> Child blocks such as `rule` in `aws_security_group` can now be dynamically generated based on lists/maps and support
> iteration. Read more at
> https://www.hashicorp.com/blog/hashicorp-terraform-0-12-preview-for-and-for-each#dynamic-nested-blocks.

Because Pulumi uses real languages, you get an incredible amount of flexibility in how to achieve this pattern,
in addition to myriad others that the HCL2 syntax doesn't support.

## Terraform 0.11

```hcl
resource "aws_autoscaling_group" "example" {
  # ...

  tag {
    key                 = "Name"
    value               = "example-asg-name"
    propagate_at_launch = false
  }

  tag {
    key                 = "Component"
    value               = "user-service"
    propagate_at_launch = true
  }

  tag {
    key                 = "Environment"
    value               = "production"
    propagate_at_launch = true
  }
}
```

## Terraform 0.12

```
locals {
  standard_tags = {
    Component   = "user-service"
    Environment = "production"
  }
}

resource "aws_autoscaling_group" "example" {
  # ...

  tag {
    key                 = "Name"
    value               = "example-asg-name"
    propagate_at_launch = false
  }

  dynamic "tag" {
    for_each = local.standard_tags

    content {
      key                 = tag.key
      value               = tag.value
      propagate_at_launch = true
    }
  }
}
```

# Pulumi

```typescript
const standardTags = [
    [ "Component", "user-service" ],
    [ "Environment", "production" ],
];
const example = new aws.ec2.AutoscalingGroup("example", {
    // ...
    tags: [{ key: "Name", value: "example-asg-name", propagateAtLaunch: true }].
        concat(standardTags.map(kvp => { key: kvp[0], value: kvp[1], propagateAtLaunch: false }));
});
```

# Generalized "Splat" Operator

> The special `resource.*.field` syntax used to only work for resources with count set. This is now a generalized
> operator that works for any list value. Read more at
> https://www.hashicorp.com/blog/terraform-0-12-generalized-splat-operator.

In Pulumi, we can easily create and manipulate first class list and map data structures, thanks to being
a general purpose language, and no new syntax is needed.

## Example: Splat Operator

### Terraform 0.12

```hcl
output "instance_ip_addrs" {
  value = google_compute_instance.example.network_interface.*.address
}
```

### Pulumi

```typescript
export const instanceIpAddrs = instance.networkInterfaces.mapApply(ni => ni.address);
```

## Example: Full Splat Operator

### Terraform 0.12

```hcl
output "instance_net_ip_addrs" {
  value = google_compute_instance.example.network_interface[*].access_config[0].assigned_nat_ip
}
```

### Pulumi

```typescript
export const instanceNetIpAddrs = instance.networkInterfaces.mapApply(ni => ni.accessConfig[0].assignedNatIp);
```

# Conditional Improvements

> The conditional operator `... ? ... : ...` now supports any value type and lazily evaluates results, as those
> familiar with this operator in other languages would expect. Also, the special value `null` can now be assigned to
> any field to represent the absence of a value. This causes Terraform to omit the field from upstream API calls,
> which is important in some cases for triggering certain default behaviors. Read more at
> https://www.hashicorp.com/blog/terraform-0-12-conditional-operator-improvements.

Pulumi supports all the usual conditional operators of languages you know and love, including ternary, traditional
`if ... else` statements, and `case` statements. Pulumi already supports `null` and `undefined` as values that
indicate the absence of a value, as our languages traditionally already do.

## Example: Conditional Operator Improvements

### Terraform 0.12

```hcl
locals {
  first_id = length(azurerm_virtual_machine.example) > 0 ? azurerm_virtual_machine.example[0].id : ""

  buckets = (var.env == "dev" ? [var.build_bucket, var.qa_bucket] : [var.prod_bucket])
}
```

### Pulumi

```typescript
const firstId = azureVirtualMachines.length ? azureVirtualMachines[0].id : "";
const buckets = env == "dev" ? [buildBucket, qaBucket] : [prodBucket];
```

## Example: Conditionally Omitted Arguments

### Terraform 0.12

```hcl
variable "override_private_ip" {
  type    = string
  default = null
}

resource "aws_instance" "example" {
  # ... (other aws_instance arguments) ...

  private_ip = var.override_private_ip
}
```

### Pulumi

```typescript
const example = new aws.ec2.Instance("example", {
    privateIp: config.get("overridePrivateIp"),
});
```

# Rich Types in Module Inputs and Outputs

> Terraform has supported basic lists and maps as inputs/outputs since Terraform 0.7, but elements were limited to
> only simple values. Terraform 0.12 allows arbitrarily complex lists and maps for any inputs and outputs, including
> with modules. Read more at https://www.hashicorp.com/blog/terraform-0-12-rich-value-types.

In Pulumi, modules take the form of custom classes whose constructors already support rich types.

## Example: Complex Values

### Terraform 0.12

```hcl
module "subnets" {
  source = "./subnets"

  parent_vpc_id = "vpc-abcd1234"
  networks = {
    production_a = {
      network_number    = 1
      availability_zone = "us-east-1a"
    }
    production_b = {
      network_number    = 2
      availability_zone = "us-east-1b"
    }
    staging_a = {
      network_number    = 1
      availability_zone = "us-east-1a"
    }
  }
}
```

### Pulumi

```typescript
import {Subnets} from "./subnets";
const subnets = new Subnets({
    parentVpcId: "vpc-abcd1234",
    networks: {
        "production_a": {
            networkNumber: 1,
            availabilityZone: "us-east-1a",
        },
        "production_b": {
            networkNumber: 2,
            availabilityZone: "us-east-1b",
        },
        "staging_a": {
            networkNumber: 1,
            availabilityZone: "us-east-1a",
        },
    },
});
```

## Example: Rich Types (1/2)

### Terraform 0.12

```hcl
variable "environment_name" {
  type = string
}
```

### Pulumi

```typescript
let environmentName: string;
```

## Example: Rich Types (2/2)

### Terraform 0.12

```hcl
variable "networks" {
  type = map(object({
    network_number    = number
    availability_zone = string
    tags              = map(string)
  }))
}
```

### Pulumi

```typescript
let networks: Record<string, {
    networkNumber: number;
    availabilityZone: string;
    tags: Record<string, string>;
}>;
```

## Example: Resources and Modules as Values

### Terraform 0.12

```hcl
output "vpc" {
  value = aws_vpc.example
}
```

### Pulumi

```typescript
const vpc = example;
```

# Template Syntax

> Within string values, a new template syntax can be used for looping without complex nested interpolation. Example:
> `%{ for instance in aws_instance.example ~}server ${instance.id}%{ endfor }`. Read more at
> https://www.hashicorp.com/blog/terraform-0-12-template-syntax.

Modern languages support various was of creating strings, including interpolation, literal strings, in addition
to dynamic concatenation which, when mixed with all the features of a general purpose language, make strings quite
easy to construct out of pieces of other program information.

## Terraform 0.12

```hcl
locals {
  lb_config = <<EOT
%{ for instance in opc_compute_instance.example ~}
server ${instance.label} ${instance.ip_address}:8080
%{ endfor }
EOT
}
```

## Pulumi

```typescript
const lbConfig = opcComputeInstances.map(i => `server ${i.label} ${i.ipAddress}:8080`).join();
```

# Reliable JSON Syntax

> Terraform 0.12 HCL configuration has an exact 1:1 mapping to and from JSON. Read more at
> https://www.hashicorp.com/blog/terraform-0-12-reliable-json-syntax.

Pulumi uses real languages which have great support for JSON, whether that is the built-in support in
JavaScript and TypeScript, or by using a library package for Python, Go, or others.

# References as First-Class Values

> References to resources and modules for fields such as depends_on used to be arbitrary strings. In Terraform 0.12,
> the resource identifier can be used exactly such as `aws_kms_grant.example` (no quotes!). This improves the validation
> and error messages we can provide. Similarly, a resource reference can be returned from a module as an output or
> accepted as a parameter.

In Pulumi, all objects are just objects, and so enjoy all the usual benefits, in the same ways, of ordinary variables.
