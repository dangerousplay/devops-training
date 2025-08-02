### **3 - Authoring Vault Policies for Access Control**

**🎯 Lesson Objective:**

By completing these exercises, you will be able to:

1.  **Experience** Vault's "deny by default" security posture.
2.  **Write, upload, and read** a basic HCL policy file.
3.  **Create a token** with limited permissions and test its boundaries.
4.  **Use wildcards** to create more flexible, yet scoped, policies.
5.  **Use Vault itself** to automatically generate the policy needed for a specific command.

-----

### **Prerequisites**

- Read the [vault policy guide](https://github.com/jeffsanicola/vault-policy-guide) 
- Read the [official vault policies tutorial](https://developer.hashicorp.com/vault/tutorials/policies/policies#policies)

### **Setup: Preparing Your Lab Environment**

1.  Start your Vault dev server:
    ```shell
    vault server -dev
    ```
2.  Copy the **Root Token** from the output.
3.  In a **new terminal window**, set up your environment variables:
    ```shell
    export VAULT_ADDR='http://127.0.0.1:8200'
    export VAULT_TOKEN='<PASTE_THE_ROOT_TOKEN_HERE>'
    ```
4.  Confirm your connection with `vault status`.

-----

### **Lab Exercises**

#### **Exercise 1: Experiencing "Deny by Default"**

Vault's default policy allows a token to manage itself, but nothing else. Let's see what that means.

1.  **Command:** Create a new token that only has the `default` policy attached.
    ```shell
    vault token create -policy="default" -ttl="10m"
    ```
2.  **Action:** Copy the new token value and set it as your `VAULT_TOKEN`.
    ```shell
    export VAULT_TOKEN='<PASTE_THE_NEW_TOKEN_HERE>'
    ```
3.  **Command:** Now, try to store a test secret.
    ```shell
    vault kv put secret/test message="hello"
    ```
4.  **Observation:** The command fails with a **permission denied** error.

> **❓ Question:** Why did this command fail even though you are authenticated with a valid token? What does this demonstrate about Vault's fundamental security model?

-----

#### **Exercise 2: Writing and Applying Your First Policy**

Let's create a policy that grants the absolute minimum permission needed: read-only access to a single secret path.

1.  **Action:** First, switch back to your powerful root token to act as an administrator.
    ```shell
    export VAULT_TOKEN='<PASTE_THE_ROOT_TOKEN_HERE>'
    ```
2.  **Action:** Create a new file named `kv-reader-policy.hcl`.
3.  **Action:** Add the following HCL code to the file. This policy grants the `read` capability to the path where our test secret will live.
    ```hcl
    # Grant read-only access to the 'test' secret path
    path "secret/data/test" {
      capabilities = ["read"]
    }
    ```
4.  **Command:** Upload this policy to Vault under the name `kv-reader`.
    ```shell
    vault policy write kv-reader kv-reader-policy.hcl
    ```
5.  **Command:** Read the policy back from Vault to confirm it was uploaded correctly.
    ```shell
    vault policy read kv-reader
    ```
> **❓ Question:** As mentioned in the documentation, why do we have to specify `secret/data/` in the policy path when we use `kv put secret/...` in the command?

-----

#### **Exercise 3: Testing Policy Boundaries**

Now let's create a token with our new policy and see exactly what it can and cannot do.

1.  **Command:** Create a new token that is only associated with the `kv-reader` policy.
    ```shell
    vault token create -policy="kv-reader"
    ```
2.  **Action:** Export this new token as your `VAULT_TOKEN`.
3.  **Action:** First, let's make sure there's a secret to read. Switch back to your **root token** and run: `vault kv put secret/test message="this is a test"`. Then, switch back to your **kv-reader token**.
4.  **Command (Should Succeed):** Try to read the secret at the allowed path.
    ```shell
    vault kv get secret/test
    ```
5.  **Command (Should Fail):** Try to write a new value to the same path.
    ```shell
    vault kv put secret/test message="trying to overwrite"
    ```
6.  **Command (Should Fail):** Try to read a secret from a different path.
    ```shell
    vault kv get secret/another-path
    ```

> **❓ Question:** Explain precisely why the first command succeeded but the next two failed by referencing the specific `path` and `capabilities` defined in your `kv-reader-policy.hcl` file.

-----

#### **Exercise 4: Using Wildcards for Broader Permissions**

Sometimes an application needs to manage all secrets under a specific prefix. This is where the wildcard (`*`) is useful.

1.  **Action:** Switch back to your **root token**.
2.  **Action:** Create a new policy file named `app-admin-policy.hcl`. This policy will grant full CRUDL (Create, Read, Update, Delete, List) permissions to everything under the `secret/cookiebot/` path.
    ```hcl
    # Grant full control over all secrets for the Cookiebot app
    path "secret/data/cookiebot/*" {
      capabilities = ["create", "read", "update", "delete", "list"]
    }
    ```
3.  **Command:** Upload this new policy to Vault.
    ```shell
    vault policy write app-admin app-admin-policy.hcl
    ```
4.  **Command:** Create a new token with the `app-admin` policy and export it as your `VAULT_TOKEN`.
5.  **Action:** Now, test your new powers\! Try running `vault kv put`, `get`, `list`, and `delete` commands on paths like `secret/cookiebot/database` and `secret/cookiebot/telegram`. Then, try to access `secret/other-app`.

> **❓ Question:** How does the wildcard (`*`) change the scope of this policy compared to the `kv-reader` policy? What is a potential security risk of using wildcards too broadly (e.g., `secret/*`)?

-----

#### **Exercise 5: The `-output-policy` Superpower**

Creating complex policies can be difficult. 
Luckily, Vault can tell you exactly what policy is needed to perform a specific action.

1.  **Action:** Switch back to your **root token**.
2.  **Command:** Let's see what policy is needed to enable the `userpass` auth method. The `-output-policy` flag prints the policy without actually running the command.
    ```shell
    vault auth enable -output-policy userpass
    ```
3.  **Observation:** Vault prints a perfect, ready-to-use HCL policy to the screen.
4.  **Command:** Now try it with a secrets engine command.
    ```shell
    vault secrets enable -output-policy -path=pki pki
    ```

> **❓ Question:** How can this `-output-policy` flag dramatically speed up your workflow when you need to create a new, least-privilege policy for an operator or an application?

#### **Exercise 6: The Power of `deny` - Precedence Rules**

**Objective:** Understand that an explicit `deny` capability always overrides any `allow` capabilities.

**Scenario:** An application has a general, read-only role for most secrets. 
However, there is a specific, highly sensitive path (`secret/financials/`) that this role must *never* be able to access, even by accident.

1.  **Action:** Create a policy file named `read-all-apps-policy.hcl`. This policy is very broad.
    ```hcl
    # Grant read access to all application secrets
    path "secret/data/apps/*" {
      capabilities = ["read"]
    }
    ```
2.  **Action:** Create a second policy file named `deny-financials-policy.hcl`. This policy is very specific.
    ```hcl
    # Explicitly deny all access to the financials path
    path "secret/data/apps/financials" {
      capabilities = ["deny"]
    }
    ```
3.  **Command:** Write both policies to Vault.
    ```shell
    vault policy write read-all-apps read-all-apps-policy.hcl
    vault policy write deny-financials deny-financials-policy.hcl
    ```
4.  **Command:** Now, create a single token that has **both** policies attached.
    ```shell
    vault token create -policy="read-all-apps" -policy="deny-financials"
    ```
5.  **Action:** Export this new token as your `VAULT_TOKEN`.
6.  **Action:** First, put some secrets in place using your **root token**.
    ```shell
    # (Use Root Token)
    vault kv put secret/apps/cookiebot api_key=123
    vault kv put secret/apps/financials credit_card=456
    ```
7.  **Action:** Now, switch back to your **new token with two policies**.
8.  **Command (Should Succeed):** Try to read the general app secret.
    ```shell
    vault kv get secret/apps/cookiebot
    ```
9.  **Command (Should Fail):** Try to read the financial secret.
    ```shell
    vault kv get secret/apps/financials
    ```

> **❓ Question:** Even though the `read-all-apps` policy granted access to `secret/data/apps/*`, you couldn't read the financials secret. What does this experiment prove about how Vault processes multiple policies and the `deny` capability?

-----

#### **Exercise 7: Controlling Sub-paths with Path Globbing (`+`)**

**Objective:** Use the `+` wildcard to grant access to any path nested under a specific prefix.

**Scenario:** The DevOps team needs full control over all secrets stored under the `ops/` prefix, including any sub-folders created in the future (e.g., `ops/terraform/gcp`, `ops/ansible/keys`).

1.  **Action:** Switch back to your **root token**.
2.  **Action:** Create a policy file named `ops-team-policy.hcl`. The `+` wildcard matches any number of characters in a single path segment.
    ```hcl
    # Grant list capability at the ops/ folder
    path "secret/metadata/ops/" {
      capabilities = ["list"]
    }

    # Grant full capabilities to any secret inside ops/ and any sub-path
    path "secret/data/ops/+" {
      capabilities = ["create", "read", "update", "delete", "list"]
    }
    ```
3.  **Command:** Write the policy to Vault and create a new token with it. Export that token.
4.  **Action:** Now, test the boundaries with your new `ops-team` token.
5.  **Command (Should Succeed):**
    ```shell
    vault kv put secret/ops/terraform/gcp/service-account key=...
    vault kv put secret/ops/ansible/ssh-key key=...
    vault kv list secret/ops/
    ```
6.  **Command (Should Fail):** Try to access a path outside the `ops/` prefix.
    ```shell
    vault kv put secret/dev/app-key key=...
    ```

> **❓ Question:** In your own words, what is the main difference between using a `*` wildcard (like `ops/*`) versus a `+` wildcard (like `ops/+`) in a policy path? When would you choose one over the other?

-----

#### **Exercise 8: Delegating Policy Administration**

**Objective:** Create a policy for a "Team Lead" role that can manage other application policies but is blocked from touching critical system policies.

**Scenario:** We want to empower team leads to create and manage policies for their own applications (e.g., `cookiebot-policy`, `invoicing-app-policy`), but they must not be able to view or change the `root`, `default`, or their own `team-lead` policy.

1.  **Action:** Switch back to your **root token**.
2.  **Action:** Create a file named `team-lead-policy.hcl`. 
This policy uses a combination of `allow` and `deny` rules on the `sys/policies/acl` path.
    ```hcl
    # Allow full management of any policy
    path "sys/policies/acl/*" {
      capabilities = ["create", "read", "update", "delete", "list"]
    }

    # But, explicitly deny access to critical policies
    path "sys/policies/acl/root" { capabilities = ["deny"] }
    path "sys/policies/acl/default" { capabilities = ["deny"] }
    path "sys/policies/acl/team-lead" { capabilities = ["deny"] }
    ```
3.  **Command:** Write the `team-lead` policy and create a new token with it. Export the new token.
4.  **Action:** Test your delegated powers with the `team-lead` token.
5.  **Command (Should Succeed):** Create a new policy for a fictional application.
    ```shell
    vault policy write new-app-policy -<<EOF
    path "secret/data/new-app/*" { capabilities = ["read"] }
    EOF
    ```
6.  **Command (Should Fail):** Try to read the powerful `root` policy.
    ```shell
    vault policy read root
    ```
7.  **Command (Should Fail):** Try to delete your own policy.
    ```shell
    vault policy delete team-lead
    ```

> **❓ Question:** Why is it a critical security practice to pair broad `allow` rules (like `sys/policies/acl/*`) with specific `deny` rules when creating administrative roles? 
