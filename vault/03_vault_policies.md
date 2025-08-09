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

-----

### **A Note on KV Secrets Engines: Version 1 vs. Version 2**

Throughout these exercises, you will see references to `secret/data/...` in the policy paths. This is a crucial detail related to the **version** of the Key-Value (KV) secrets engine being used.

- **KV Version 2 (KV v2):** This is the **default** version enabled when you run `vault server -dev`. It is a **versioned** key-value store, meaning it keeps a history of previous versions of your secrets. Because of this versioning feature, the API paths are segmented.
    - To access the secret data itself, you must use the `secret/data/...` path in your policy.
    - To perform operations on the secret's metadata (like listing versions or deleting the secret entirely), you use the `secret/metadata/...` path.

- **KV Version 1 (KV v1):** This is the original, **unversioned** key-value store. It is simpler: when you write a new secret to a path, it overwrites the previous one.
    - Policy paths correspond directly to the mount path, like `kv/my-secret`. There is no `data/` or `metadata/` infix required.

The `vault kv ...` commands intelligently interact with the correct API endpoint based on the version of the engine at the target path.
The following exercises will show you how to work with a KV v1 engine.

-----

#### **Exercise 9: Enabling and Authoring for KV Version 1**

**Objective:** Enable a KV v1 secrets engine and write a policy for it, noting the difference in the policy path structure.

**Scenario:** A legacy application requires a non-versioned key-value store. 
We will enable a KV v1 engine at the path `kv-legacy/` and create a policy that grants full access to it.

1.  **Action:** Switch back to your **root token**.
2.  **Command:** First, enable the KV v1 secrets engine at a new path.
    ```shell
    vault secrets enable -path=kv-legacy -version=1 kv
    ```
3.  **Action:** Create a policy file named `legacy-app-policy.hcl`. Note the simple path structure without `data/`.
    ```hcl
    # Grant full capabilities to the legacy KV store
    path "kv-legacy/*" {
      capabilities = ["create", "read", "update", "delete", "list"]
    }
    ```
4.  **Command:** Write the policy and create a token with it.
    ```shell
    vault policy write legacy-app legacy-app-policy.hcl
    vault token create -policy="legacy-app"
    ```
5.  **Action:** Export the new token as your `VAULT_TOKEN`.
6.  **Action:** Test your access to the KV v1 store.
7.  **Command (Should Succeed):**
    ```shell
    vault kv put kv-legacy/config/database-url value="127.0.0.1"
    vault kv get kv-legacy/config/database-url
    vault kv list kv-legacy/config/
    ```
8.  **Command (Should Fail):** Try to access the default KV v2 store.
    ```shell
    vault kv get secret/test
    ```

> **❓ Question:** Look at the `path` in `legacy-app-policy.hcl`. Why is `data/` not required in this path, unlike the policies for the `secret/` path used in earlier exercises?

-----

#### **Exercise 10: KV v1 `list` Permissions**

**Objective:** Understand how the `list` capability functions in a KV v1 policy.

**Scenario:** An automated script needs to discover all secrets under a specific prefix in the `kv-legacy` store, but it must not be allowed to read the secret values themselves.

1.  **Action:** Switch back to your **root token**.
2.  **Action:** Make sure there are a few secrets to find in the `kv-legacy` store.
    ```shell
    # (Use Root Token)
    vault kv put kv-legacy/inventory/server-01 ip=10.0.0.1
    vault kv put kv-legacy/inventory/server-02 ip=10.0.0.2
    ```
3.  **Action:** Create a new policy file named `kv-lister-policy.hcl`. This policy only grants the `list` capability.
    ```hcl
    # Grant only list capability to the inventory path
    path "kv-legacy/inventory/*" {
      capabilities = ["list"]
    }
    ```
4.  **Command:** Write the policy to Vault and create a token for it.
    ```shell
    vault policy write kv-lister kv-lister-policy.hcl
    vault token create -policy="kv-lister"
    ```
5.  **Action:** Export the new token as your `VAULT_TOKEN`.
6.  **Command (Should Succeed):** Try to list the secrets. Note that listing a "folder" in KV requires the `list` command on that path itself.
    ```shell
    vault kv list kv-legacy/inventory/
    ```
7.  **Command (Should Fail):** Now, try to read one of the secrets you just discovered.
    ```shell
    vault kv get kv-legacy/inventory/server-01
    ```

> **❓ Question:** In KV v2, a `list` operation requires permissions on the `secret/metadata/...` path. How does the `list` permission in this KV v1 policy differ, and why is this an important distinction to remember when working with both versions?

-----

#### **Exercise 11: The Destructive Nature of KV v1 (Overwrite)**

**Objective:** Witness firsthand that KV v1 does not version secrets and an `update` overwrites data permanently.

**Scenario:** An operator needs to update an API key stored in the `kv-legacy` engine. We will observe how the old value is immediately and permanently replaced.

1.  **Action:** Switch to your **root token**.
2.  **Command:** Create an initial secret to work with.
    ```shell
    vault kv put kv-legacy/api/billing-service key="v1_initial_key_12345"
    ```
3.  **Action:** Create a policy `kv-updater-policy.hcl` that allows writing. The `vault kv put` command requires `create` for new paths and `update` for existing paths.
    ```hcl
    path "kv-legacy/api/*" {
      capabilities = ["read", "create", "update"]
    }
    ```
4.  **Command:** Write this policy and create a token with it. Export the new token.
    ```shell
    vault policy write kv-updater kv-updater-policy.hcl
    vault token create -policy="kv-updater"
    ```
5.  **Command (Read):** First, read the secret to see the original value.
    ```shell
    vault kv get kv-legacy/api/billing-service
    ```
6.  **Command (Update):** Now, `put` a new value in the same path.
    ```shell
    vault kv put kv-legacy/api/billing-service key="v2_updated_key_ABCDE"
    ```
7.  **Command (Confirm):** Read the secret again. You will see the new value. With KV v1, the old value is gone forever.
    ```shell
    vault kv get kv-legacy/api/billing-service
    ```

> **❓ Question:** If this had been a KV v2 secrets engine, how would the outcome of step 6 have been different? What command could you have used to retrieve the original key ("v1_initial_key_12345")?

-----

#### **Exercise 12: Separating `create` and `update` Capabilities in KV v1**

**Objective:** Implement a "write-once" policy by differentiating between the `create` and `update` capabilities.

**Scenario:** We need to provision initial credentials for new applications. The provisioning process should be able to write a secret once, but it should not be allowed to modify it later.

1.  **Action:** Switch to your **root token**.
2.  **Action:** Create a policy file `write-once-policy.hcl`. This policy *only* includes the `create` capability.
    ```hcl
    # Allow creating, but not updating, secrets in the app-init path
    path "kv-legacy/app-init/*" {
      capabilities = ["create", "read"]
    }
    ```
3.  **Command:** Write the policy and create a token. Export the new token.
    ```shell
    vault policy write write-once kv-once-policy.hcl
    vault token create -policy="write-once"
    ```
4.  **Command (Should Succeed):** Use your new token to write a secret to a brand new path. This works because the path doesn't exist yet, so the action is a `create`.
    ```shell
    vault kv put kv-legacy/app-init/new-app-123 token="initial-bootstrap-token"
    ```
5.  **Command (Should Fail):** Now, immediately try to write to that *same path* again. This fails because the path now exists, and your policy does not grant the `update` capability.
    ```shell
    vault kv put kv-legacy/app-init/new-app-123 token="trying-to-change-token"
    ```

> **❓ Question:** Describe a security scenario where a "write-once" policy like this would be highly valuable for preventing accidental or malicious changes.

-----

#### **Exercise 13: Granular Control with the `+` Wildcard in KV v1**

**Objective:** Use the `+` wildcard to scope permissions to a specific level in a path hierarchy.

**Scenario:** We have a KV v1 store for infrastructure secrets. A "network-admin" role should be allowed to manage secrets for different environments (e.g., `prod`, `dev`) inside the `networking` folder, but they must not be able to manage secrets in sub-folders (like `networking/prod/internal`) or in other top-level folders (like `compute`).

1.  **Action:** Switch to your **root token** and create some placeholder secrets.
    ```shell
    # (Use Root Token)
    vault kv put kv-legacy/networking/prod/firewall-rules value="..."
    vault kv put kv-legacy/networking/dev/switch-config value="..."
    vault kv put kv-legacy/networking/prod/internal/vpn-key value="..."
    vault kv put kv-legacy/compute/prod/instance-key value="..."
    ```
2.  **Action:** Create `network-admin-policy.hcl`. The `+` wildcard in the path will match any single path segment (like `prod` or `dev`) but not multiple segments (like `prod/internal`).
    ```hcl
    # Allow managing secrets in the immediate sub-folders of 'networking'
    path "kv-legacy/networking/+/+" {
      capabilities = ["create", "read", "update", "delete", "list"]
    }
    ```
3.  **Command:** Write the policy and create a token. Export the new token.
4.  **Action:** Test your permissions with the `network-admin` token.
5.  **Command (Should Succeed):** You can read and write to paths directly under `networking/<env>/`.
    ```shell
    vault kv get kv-legacy/networking/prod/firewall-rules
    vault kv put kv-legacy/networking/staging/new-config value="new"
    ```
6.  **Command (Should Fail):** You cannot access deeper paths or paths outside of `networking`.
    ```shell
    vault kv get kv-legacy/networking/prod/internal/vpn-key
    vault kv get kv-legacy/compute/prod/instance-key
    ```

> **❓ Question:** How would the behavior of this policy change if you replaced `kv-legacy/networking/+/+` with `kv-legacy/networking/*`? Which paths that previously failed would now succeed?
