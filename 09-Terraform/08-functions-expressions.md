# Terraform Functions and Expressions

## Expression Types

Terraform supports various expression types for dynamic configurations.

```hcl
# Literal values
string_value = "hello"
number_value = 42
bool_value   = true

# References
instance_id = aws_instance.web.id
region      = var.region
prefix      = local.name_prefix

# Operators
sum    = 1 + 2
concat = "Hello, ${var.name}!"
```

---

## Conditional Expressions

### Ternary Operator

```hcl
# Syntax: condition ? true_value : false_value

# Simple conditional
instance_type = var.environment == "production" ? "t3.large" : "t3.micro"

# Nested conditional
instance_type = var.environment == "production" ? "t3.large" : (
  var.environment == "staging" ? "t3.medium" : "t3.micro"
)

# Boolean to string
status = var.enabled ? "active" : "inactive"

# Conditional resource count
resource "aws_instance" "bastion" {
  count = var.create_bastion ? 1 : 0

  ami           = var.ami_id
  instance_type = "t3.micro"
}

# Conditional value in map
tags = {
  Environment = var.environment
  Debug       = var.debug_mode ? "enabled" : "disabled"
}
```

### Conditional with null

```hcl
# Use null to omit optional arguments
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  # Only set if provided
  key_name = var.key_name != "" ? var.key_name : null

  # Using try() for optional nested values
  iam_instance_profile = try(var.instance_config.iam_profile, null)
}
```

---

## String Functions

### Basic String Functions

```hcl
# format - Printf-style formatting
formatted = format("Hello, %s! You have %d messages.", var.name, var.count)
# "Hello, John! You have 5 messages."

# formatlist - Format each item in a list
formatted_list = formatlist("user-%s", ["alice", "bob", "charlie"])
# ["user-alice", "user-bob", "user-charlie"]

# join - Concatenate list with delimiter
joined = join(", ", ["a", "b", "c"])
# "a, b, c"

# split - Split string into list
parts = split(",", "a,b,c")
# ["a", "b", "c"]

# lower / upper
lower_name = lower("HELLO")  # "hello"
upper_name = upper("hello")  # "HELLO"

# title - Capitalize first letter of each word
title_case = title("hello world")  # "Hello World"

# trim / trimprefix / trimsuffix
trimmed = trim("  hello  ", " ")       # "hello"
no_prefix = trimprefix("helloworld", "hello")  # "world"
no_suffix = trimsuffix("helloworld", "world")  # "hello"

# replace
replaced = replace("hello world", "world", "terraform")
# "hello terraform"

# substr - Substring
sub = substr("hello world", 0, 5)  # "hello"

# length
len = length("hello")  # 5
```

### Regex Functions

```hcl
# regex - Extract matching string
match = regex("[a-z]+", "123abc456")  # "abc"

# regexall - Extract all matches
matches = regexall("[a-z]+", "abc123def456")  # ["abc", "def"]

# can + regex for validation
is_valid = can(regex("^[a-z]+$", var.name))

# replace with regex
cleaned = replace("hello-123-world", "/[0-9]+/", "")  # "hello--world"
```

### String Templates

```hcl
# Heredoc
user_data = <<-EOF
  #!/bin/bash
  echo "Hello, ${var.name}!"
  apt-get update
  apt-get install -y ${join(" ", var.packages)}
EOF

# templatefile function
user_data = templatefile("${path.module}/templates/user_data.sh", {
  hostname = var.hostname
  packages = var.packages
})

# templates/user_data.sh
# #!/bin/bash
# hostnamectl set-hostname ${hostname}
# %{ for pkg in packages ~}
# apt-get install -y ${pkg}
# %{ endfor ~}
```

---

## Collection Functions

### List Functions

```hcl
# length
count = length(["a", "b", "c"])  # 3

# element - Get item by index (wraps around)
first = element(["a", "b", "c"], 0)  # "a"
wrap  = element(["a", "b", "c"], 4)  # "b" (wraps)

# index - Find index of value
idx = index(["a", "b", "c"], "b")  # 1

# contains
has_b = contains(["a", "b", "c"], "b")  # true

# concat - Merge lists
merged = concat(["a", "b"], ["c", "d"])  # ["a", "b", "c", "d"]

# flatten - Flatten nested lists
flat = flatten([["a", "b"], ["c", ["d"]]])  # ["a", "b", "c", "d"]

# compact - Remove empty strings
cleaned = compact(["a", "", "b", "", "c"])  # ["a", "b", "c"]

# distinct - Remove duplicates
unique = distinct(["a", "b", "a", "c", "b"])  # ["a", "b", "c"]

# reverse
reversed = reverse(["a", "b", "c"])  # ["c", "b", "a"]

# sort
sorted = sort(["c", "a", "b"])  # ["a", "b", "c"]

# slice - Get sublist
sub = slice(["a", "b", "c", "d"], 1, 3)  # ["b", "c"]

# range - Generate number sequence
nums = range(5)       # [0, 1, 2, 3, 4]
nums = range(1, 5)    # [1, 2, 3, 4]
nums = range(0, 10, 2) # [0, 2, 4, 6, 8]

# coalesce - First non-null, non-empty
value = coalesce("", null, "default")  # "default"

# coalescelist - First non-empty list
list = coalescelist([], ["a", "b"])  # ["a", "b"]

# one - Extract single element (or null)
single = one(["only-item"])  # "only-item"
single = one([])              # null
```

### Map Functions

```hcl
# keys / values
k = keys({a = 1, b = 2})    # ["a", "b"]
v = values({a = 1, b = 2})  # [1, 2]

# lookup - Get value with default
val = lookup({a = 1, b = 2}, "c", 0)  # 0 (default)

# merge - Combine maps
combined = merge(
  {a = 1, b = 2},
  {b = 3, c = 4}
)  # {a = 1, b = 3, c = 4}

# zipmap - Create map from two lists
map = zipmap(
  ["name", "age"],
  ["Alice", 30]
)  # {name = "Alice", age = 30}

# transpose - Swap keys and values (for maps of lists)
transposed = transpose({
  a = ["1", "2"]
  b = ["1", "3"]
})  # {1 = ["a", "b"], 2 = ["a"], 3 = ["b"]}
```

### Set Functions

```hcl
# toset - Convert to set
my_set = toset(["a", "b", "a"])  # ["a", "b"]

# setintersection - Common elements
common = setintersection(["a", "b", "c"], ["b", "c", "d"])  # ["b", "c"]

# setunion - All elements
all = setunion(["a", "b"], ["b", "c"])  # ["a", "b", "c"]

# setsubtract - Difference
diff = setsubtract(["a", "b", "c"], ["b"])  # ["a", "c"]

# setproduct - Cartesian product
product = setproduct(["a", "b"], ["1", "2"])
# [["a", "1"], ["a", "2"], ["b", "1"], ["b", "2"]]
```

---

## Numeric Functions

```hcl
# abs - Absolute value
absolute = abs(-5)  # 5

# ceil / floor - Round up/down
up   = ceil(4.3)   # 5
down = floor(4.7)  # 4

# min / max
minimum = min(1, 5, 3)  # 1
maximum = max(1, 5, 3)  # 5

# log - Logarithm
log_val = log(100, 10)  # 2

# pow - Power
power = pow(2, 3)  # 8

# signum - Sign (-1, 0, 1)
sign = signum(-5)  # -1

# parseint - Parse string to int
num = parseint("FF", 16)  # 255
```

---

## Filesystem Functions

```hcl
# file - Read file content
content = file("${path.module}/files/config.txt")

# fileexists - Check if file exists
exists = fileexists("${path.module}/optional.txt")

# fileset - List files matching pattern
files = fileset(path.module, "templates/*.tpl")

# filebase64 - Read file as base64
encoded = filebase64("${path.module}/files/image.png")

# templatefile - Process template
rendered = templatefile("${path.module}/templates/config.tpl", {
  name  = var.name
  items = var.items
})

# dirname / basename
dir  = dirname("/path/to/file.txt")   # "/path/to"
base = basename("/path/to/file.txt")  # "file.txt"

# pathexpand - Expand ~
home = pathexpand("~/.config")  # "/home/user/.config"

# abspath - Absolute path
abs = abspath("./relative/path")
```

---

## Type Conversion Functions

```hcl
# tostring / tonumber / tobool
str = tostring(42)     # "42"
num = tonumber("42")   # 42
bool = tobool("true")  # true

# tolist / toset / tomap
list = tolist(["a", "b"])
set  = toset(["a", "b", "a"])
map  = tomap({a = 1, b = 2})

# type - Get type (for debugging)
type_info = type(var.example)

# try - Return first successful expression
value = try(var.complex.nested.value, "default")

# can - Test if expression is valid
is_valid = can(tonumber(var.input))
```

---

## Date and Time Functions

```hcl
# timestamp - Current UTC time
now = timestamp()  # "2024-01-15T10:30:00Z"

# formatdate - Format timestamp
formatted = formatdate("YYYY-MM-DD", timestamp())
# "2024-01-15"

formatted = formatdate("DD MMM YYYY hh:mm AA", "2024-01-15T10:30:00Z")
# "15 Jan 2024 10:30 AM"

# timeadd - Add duration to timestamp
future = timeadd(timestamp(), "24h")
future = timeadd(timestamp(), "720h")  # 30 days

# timecmp - Compare timestamps
cmp = timecmp("2024-01-01T00:00:00Z", "2024-01-02T00:00:00Z")
# -1 (first is earlier)

# plantimestamp - Plan time (consistent across plan)
plan_time = plantimestamp()
```

---

## For Expressions

### List Transformation

```hcl
# Transform list
names = [for s in var.list : upper(s)]
# ["a", "b"] -> ["A", "B"]

# Filter list
adults = [for p in var.people : p.name if p.age >= 18]

# With index
indexed = [for i, v in var.list : "${i}: ${v}"]
# ["0: a", "1: b", "2: c"]

# Nested for
flattened = flatten([
  for group in var.groups : [
    for user in group.users : {
      group = group.name
      user  = user
    }
  ]
])
```

### Map Transformation

```hcl
# Create map from list
name_to_id = {for inst in aws_instance.web : inst.tags.Name => inst.id}
# {"web-1" = "i-123", "web-2" = "i-456"}

# Transform map
upper_tags = {for k, v in var.tags : k => upper(v)}

# Filter map
filtered = {for k, v in var.map : k => v if v != ""}

# Swap keys and values
swapped = {for k, v in var.map : v => k}

# Group by attribute
grouped = {
  for inst in aws_instance.web :
  inst.availability_zone => inst.id...
}
# {"us-east-1a" = ["i-1", "i-2"], "us-east-1b" = ["i-3"]}
```

---

## Splat Expressions

```hcl
# Get all IDs from a list of resources
# Equivalent to: [for i in aws_instance.web : i.id]
ids = aws_instance.web[*].id

# Nested splat
security_group_ids = aws_instance.web[*].vpc_security_group_ids[0]

# With for_each resources (use values())
ids = values(aws_instance.web)[*].id
```

---

## Dynamic Blocks

```hcl
# Dynamic nested blocks
variable "ingress_rules" {
  default = [
    { port = 80, cidr = "0.0.0.0/0" },
    { port = 443, cidr = "0.0.0.0/0" },
  ]
}

resource "aws_security_group" "web" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = [ingress.value.cidr]
    }
  }
}

# Dynamic with iterator name
dynamic "ingress" {
  for_each = var.ingress_rules
  iterator = rule  # Custom iterator name

  content {
    from_port   = rule.value.port
    to_port     = rule.value.port
    protocol    = "tcp"
    cidr_blocks = [rule.value.cidr]
  }
}

# Conditional dynamic block
dynamic "logging" {
  for_each = var.enable_logging ? [1] : []

  content {
    target_bucket = var.log_bucket
    target_prefix = "logs/"
  }
}
```

---

## Summary

| Category | Functions |
|----------|-----------|
| **String** | format, join, split, replace, trim, lower, upper |
| **Collection** | length, concat, flatten, merge, lookup, keys, values |
| **Numeric** | abs, ceil, floor, min, max |
| **Filesystem** | file, fileexists, templatefile |
| **Type** | tostring, tolist, tomap, try, can |
| **Date/Time** | timestamp, formatdate, timeadd |

### Expression Types

| Expression | Syntax |
|------------|--------|
| **Conditional** | `condition ? true : false` |
| **For (list)** | `[for x in list : transform]` |
| **For (map)** | `{for k, v in map : k => v}` |
| **Splat** | `resource[*].attribute` |
| **Dynamic** | `dynamic "block" { for_each = ... }` |
