# JinjaForm Extension Test Cases

This document describes test cases for validating complex nested Jinja within Terraform syntax highlighting.

## Test Case 1: Basic Jinja + Terraform
**File**: `main.tf.j2`
**Purpose**: Baseline test with simple conditional
**Expected**: Jinja keywords and Terraform keywords should both highlight

```terraform
{% if condition %}
resource "aws_vpc" "main" {
  cidr_block = "{{ vpc_cidr }}"
}
{% endif %}
```

**Validation**:
- ✓ `{% if %}` highlighted as Jinja control flow
- ✓ `resource` highlighted as Terraform keyword
- ✓ `{{ vpc_cidr }}` highlighted as Jinja interpolation
- ✓ `{% endif %}` highlighted as Jinja control flow

---

## Test Case 2: Complex Nested Structures
**File**: `complex_nested.tf.j2`
**Purpose**: Test deeply nested Jinja within Terraform resource blocks
**Complexity**: Multiple levels of nesting with:
- Jinja conditionals wrapping Terraform resources
- Jinja loops inside Terraform resource blocks (tags, ingress rules)
- Nested Jinja loops (double nesting)
- Mixed Jinja interpolation within HCL values

### Scenario 1: Conditional Resource Block
```terraform
{% if enable_vpc %}
resource "aws_vpc" "main" {
  cidr_block = "{{ vpc_cidr }}"
}
{% endif %}
```

**Current Status**: 🟡 PARTIAL
- ✓ Outer Jinja conditional highlights
- ⚠️ Inner resource definition may lose some highlighting
- ⚠️ Variable interpolation `{{ vpc_cidr }}` inside may not highlight correctly

### Scenario 2: Jinja Loop Generating Resources
```terraform
{% for subnet in subnets %}
resource "aws_subnet" "subnet_{{ loop.index0 }}" {
  cidr_block = "{{ subnet.cidr }}"
}
{% endfor %}
```

**Current Status**: 🟡 PARTIAL
- ✓ Loop keywords highlight
- ⚠️ Resource name with Jinja interpolation `subnet_{{ loop.index0 }}` may not highlight correctly
- ⚠️ Nested interpolation in values inconsistent

### Scenario 3: Jinja Inside HCL Blocks
```terraform
resource "aws_security_group" "app" {
  {% if enable_http %}
  ingress {
    from_port   = 80
    cidr_blocks = [
      {% for cidr in allowed_cidrs %}
      "{{ cidr }}",
      {% endfor %}
    ]
  }
  {% endif %}
}
```

**Current Status**: 🔴 BROKEN
- ✗ Jinja blocks inside HCL attribute values don't parse correctly
- ✗ List items with Jinja loops fail to highlight properly
- ✗ Closing `}` may not match opening properly

### Scenario 4: Deeply Nested Jinja
```terraform
{% for env in environments %}
  {% if env.enable_db %}
resource "aws_db_instance" "{{ env.name }}" {
  # ...
}
  {% endif %}
{% endfor %}
```

**Current Status**: 🔴 BROKEN
- ✗ Nested Jinja control structures lose context
- ✗ Variable access (`env.name`) inside interpolation may fail

### Scenario 5: Jinja in Locals/Outputs
```terraform
locals {
  common_tags = {
    {% if include_owner %}
    Owner = "{{ owner_email }}"
    {% endif %}
  }

  {% if multi_region %}
  region_map = {
    {% for region_name, region_config in regions.items() %}
    "{{ region_name }}" = "{{ region_config.zone }}"
    {% endfor %}
  }
  {% else %}
  region_map = { primary = "{{ primary_region }}" }
  {% endif %}
}
```

**Current Status**: 🟡 PARTIAL
- ✓ Simple tags highlight
- ⚠️ Complex conditionals and loops in attribute values problematic
- ⚠️ Nested conditionals with fallback branches fail

---

## Test Case 3: Edge Cases

### Mixed Indentation Levels
```terraform
resource "aws_instance" "example" {
  tags = {
    {% for key, value in var.tags.items() %}
    "{{ key }}" = "{{ value }}"
    {% endfor %}
  }
}
```

**Expected**: Should handle arbitrary indentation

### Empty/Whitespace Control
```terraform
{%- if condition %}
resource "aws_resource" "name" {}
{%- endif %}
```

**Expected**: Whitespace control markers (`-`) should not break parsing

### Complex Jinja Expressions
```terraform
resource "aws_{{ resource_type }}_{{ suffix }}" "main" {
  count = {% if var.enable %}1{% else %}0{% endif %}
}
```

**Expected**: Complex interpolation patterns should work

---

## Validation Checklist

When testing syntax highlighting, verify:

### Jinja Elements
- [ ] `{% if %}` / `{% endif %}` highlighted as keyword
- [ ] `{% for %}` / `{% endfor %}` highlighted as keyword
- [ ] `{{ variable }}` highlighted as Jinja interpolation
- [ ] `{{ variable | filter }}` filter syntax highlighted
- [ ] `{# comment #}` highlighted as comment
- [ ] Whitespace control (`{%-`, `-%}`) recognized
- [ ] Nested Jinja blocks maintain consistent highlighting

### Terraform Elements
- [ ] `resource` / `data` keywords highlighted
- [ ] `provider` / `module` / `variable` / `output` / `locals` keywords highlighted
- [ ] HCL strings `"value"` highlighted
- [ ] Numbers and booleans highlighted
- [ ] Comments `#` highlighted
- [ ] Block markers `{` `}` matched correctly

### Mixed Elements
- [ ] Jinja inside resource names: `"{{ name }}"`
- [ ] Jinja inside string values: `"prefix-{{ var }}"`
- [ ] Jinja inside numeric contexts: `count = {{ expr }}`
- [ ] Jinja inside lists: `cidr_blocks = [{{ list }}]`
- [ ] Multiple Jinja elements per line
- [ ] Jinja loops over HCL blocks (resources, ingress, etc.)

### Context Preservation
- [ ] Outer Terraform context not lost by inner Jinja
- [ ] Inner Jinja context not lost by outer Terraform
- [ ] Bracket matching works across Jinja boundaries
- [ ] Code folding works for both Jinja and Terraform blocks
- [ ] Comments toggle correctly with Cmd/Ctrl + /

---

## Known Issues Reference

From README.md Known Issues section:

1. **Nested jinja blocks will not respect highlighting** ← This test case validates this issue
2. No language server support (by design)
3. Commenting uses Terraform style (by design)

---

## Testing in VSCode

### Step-by-Step Manual Testing

1. **Open `complex_nested.tf.j2` in VSCode**
   ```bash
   code examples/complex_nested.tf.j2
   ```

2. **Verify Language Detection**
   - Bottom-right corner should show "jinjaform"
   - If not, Cmd/Ctrl + K, Cmd/Ctrl + M to select language

3. **Use Inspector Tool**
   - Cmd/Ctrl + Shift + P
   - Type: "Developer: Inspect Editor Tokens and Scopes"
   - Click on various elements to see their scope names
   - Expected scopes: `source.jinja.*`, `source.hcl.terraform.*`, `source.jinjaform`

4. **Check Specific Problem Areas**
   - Lines with nested Jinja (around line 35-42 with tag loops)
   - Lines with Jinja inside HCL blocks (around line 50-75)
   - Lines with deeply nested loops (around line 85-100)

5. **Visual Indicators of Problems**
   - 🔴 RED: Keywords not colored consistently
   - 🟡 YELLOW: Mixed or muted coloring that seems wrong
   - ✓ GREEN: Proper distinct coloring for different element types

---

## Regression Testing

After grammar changes, verify:

1. All old test cases still pass (main.tf.j2)
2. No new syntax errors introduced
3. Folding still works for both Jinja and Terraform
4. Auto-completion and bracket matching unchanged
5. No performance degradation
