## 2. Create Vault policies

### Exam Objectives
- 2a. Illustrate the value of Vault policy
- 2b. Describe Vault policy syntax: `path` 
- 2c. Describe Vault policy syntax: `capabilities`
- 2d. Craft a Vault policy based on requirements

### Notes

#### Policies

Policies provide a declarative way to grant or forbid access to certain paths and operations in Vault. 
Policies are `deny by default`, so an empty policy grants no permission in the system.

#### Policy-Authorization Workflow

1. The security team authors a policy (or uses an existing policy) which grants access to paths in Vault. Policies are written in HCL.
2. The policy's contents are uploaded and stored in Vault and referenced by name. You can think of the policy's name as a pointer or symlink to its set of rules.
3. Most importantly, the security team maps data in the auth method to a policy. For example an LDAP OU group.
4. Using the mapping configured by the security team in the previous section, Vault then generates a token and attaches the matching policies.

#### Policy Syntax

Policies are written in HCL or JSON and describe which paths in Vault a user or machine is allowed to access.

Here is a very simple policy which grants read capabilities to the path `"secret/foo"`:

```bash
path "secret/foo" {
  capabilities = ["read"]
}
```
When this policy is assigned to a token, the token can read from "secret/foo". However, the token cannot update or delete "secret/foo", since the capabilities do not allow it. Because policies are deny by default, the token would have no other access in Vault.

##### More Detailed Policies

```bash
# This section grants all access on "secret/*". Further restrictions can be
# applied to this broad policy, as shown below.
path "secret/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# Even though we allowed secret/*, this line explicitly denies
# secret/super-secret. This takes precedence.
path "secret/super-secret" {
  capabilities = ["deny"]
}

# Policies can also specify allowed, disallowed, and required parameters. Here
# the key "secret/restricted" can only contain "foo" (any value) and "bar" (one
# of "zip" or "zap").
path "secret/restricted" {
  capabilities = ["create"]
  allowed_parameters = {
    "foo" = []
    "bar" = ["zip", "zap"]
  }
}

# Permit reading only "secret/foo". An attached token cannot read "secret/food"
# or "secret/foo/bar".
path "secret/foo" {
  capabilities = ["read"]
}

# Permit reading everything under "secret/bar". An attached token could read
# "secret/bar/zip", "secret/bar/zip/zap", but not "secret/bars/zip".
path "secret/bar/*" {
  capabilities = ["read"]
}

# Permit reading everything prefixed with "zip-". An attached token could read
# "secret/zip-zap" or "secret/zip-zap/zong", but not "secret/zip/zap
path "secret/zip-*" {
  capabilities = ["read"]
}
```

In addition, a `+` can be used to denote any number of characters bounded within a single path segment (this appeared in Vault 1.1):

```bash
# Permit reading the "teamb" path under any top-level path under secret/
path "secret/+/teamb" {
  capabilities = ["read"]
}

# Permit reading secret/foo/bar/teamb, secret/bar/foo/teamb, etc.
path "secret/+/+/teamb" {
  capabilities = ["read"]
}
```

> The policy rules that Vault applies are determined by the most-specific match available, using the priority rules described below. This may be an exact match or the longest-prefix match of a glob. If the same pattern appears in multiple policies, we take the union of the capabilities. If different patterns appear in the applicable policies, we take only the highest-priority match from those policies.

#### Pattern Matching Criteria

1. If the first wildcard (`+`) or glob (`*`) occurs earlier in `P1`, `P1` is lower priority
2. If `P1` ends in `*` and `P2` doesn't, `P1` is lower priority
3. If `P1` has more `+` (wildcard) segments, `P1` is lower priority
4. If `P1` is shorter, it is lower priority
5. If `P1` is smaller lexicographically, it is lower priority

> The glob character referred to in this documentation is the asterisk (`*`). It is not a regular expression and is only supported **as the last character of the path!**

#### Capabilities

Each path must define one or more capabilities which provide fine-grained control over permitted (or denied) operations

The list of capabilities are (mapped to HTTP verbs):
- `create` (`POST`/`PUT`) - Allows creating data at the given path. Very few parts of Vault distinguish between create and update, so most operations require both create and update capabilities. Parts of Vault that provide such a distinction are noted in documentation.

- `read` (`GET`) - Allows reading the data at the given path.

- `update` (`POST`/`PUT`) - Allows changing the data at the given path. In most parts of Vault, this implicitly includes the ability to create the initial value at the path.

- `delete` (`DELETE`) - Allows deleting the data at the given path.

- `list` (`LIST`) - Allows listing values at the given path. Note that the keys returned by a list operation are not filtered by policies. Do not encode sensitive information in key names. Not all backends support listing.

In addition to the standard set, there are some capabilities that do not map to HTTP verbs.

- `sudo` - Allows access to paths that are root-protected. Tokens are not permitted to interact with these paths unless they have the sudo capability (in addition to the other necessary capabilities for performing an operation against that path, such as read or delete).

For example, modifying the audit log backends requires a token with sudo privileges.

- `deny` - Disallows access. This always takes precedence regardless of any other defined capabilities, including `sudo`.
