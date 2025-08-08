### 1 - Introduction to Secrets Management with HashiCorp Vault

**🎯 Lesson Objective:**

By the end of this lesson, you will have a solid understanding of Vault's core purpose and will be able to:

1.  Start a local Vault server in "dev" mode for safe learning.
2.  Securely store a secret for the Cookiebot application.
3.  Define a restrictive access policy based on the **Principle of Least Privilege**.
4.  Create a new, limited token with that policy attached.
5.  Test the token's permissions to see the policy in action.

-----

### Overview

* **Why Use Vault?** [HashiCorp Developer](https://developer.hashicorp.com/vault/tutorials/get-started/why-use-vault)
* **Video Overview (Portuguese):** Watch the first 27 minutes of [Webinar - Usando o Hashicorp Vault](https://www.youtube.com/watch?v=eD3kN6tR2rc) to get a great conceptual overview.

### **1. Starting the Vault Dev Server**

Install hashicorp vault on your machine by following the [Vault Install official page](https://developer.hashicorp.com/vault/install)

A "dev" server is a special mode for Vault that runs entirely in-memory. 
It's perfect for learning and testing because it starts up quickly and doesn't require complex setup. 
**It should never be used for production.**

**Your Exercise:**
1.  Open your terminal and run the following command:
    ```bash
    vault server -dev
    ```
2.  Vault will start and print out a lot of information. Look for two critical pieces of information and save them in a temporary text file:
    * **Unseal Key:** Not needed for dev mode, but crucial for production.
    * **Root Token:** This is the master key to your Vault. It has permission to do *anything*.

> 📚 For more information about dev mode: ["Dev" server mode](https://developer.hashicorp.com/vault/docs/concepts/dev-server)

-----

### **2. Connecting Your Terminal to Vault**

For the `vault` command-line tool to work, it needs to know the server's address and which token to use for authentication. 
We'll set these as environment variables.

**Your Exercise:**

1.  Open a **new terminal window** (leaving the server running in the first one).
2.  Set the server address:
    ```shell
    export VAULT_ADDR='http://127.0.0.1:8200'
    ```
3.  Set the token. Copy the **Root Token** from the previous step and use it here:
    ```shell
    export VAULT_TOKEN='s.xxxxxxxxxxxxxxxxxxxx'
    ```
4.  Verify your connection:
    ```shell
    vault status
    ```
    You should see information confirming that Vault is running and unsealed.

-----

### **3. Storing Your First Secret**

Vault stores secrets in different "Secrets Engines." 
The default one is the Key/Value (KV) engine, which works like a secure dictionary. 
Let's store the Telegram token for Cookiebot.

**Your Exercise:**

1.  Use the `vault kv put` command to store the secret at the path `secret/app/cookiebot`:
    ```shell
    vault kv put secret/app/cookiebot telegram_token="super-secret-12345"
    ```
2.  Read the secret back to confirm it was stored correctly:
    ```shell
    vault kv get secret/app/cookiebot
    ```
    You should see the key (`telegram_token`) and its value in the output.

-----

### **4. Creating a Policy: The Rulebook**

We never want to give our applications the all-powerful root token. 
Instead, we follow the **Principle of Least Privilege**: grant only the absolute minimum permissions required for a task. 
We define these permissions in a **policy**.

**Your Exercise:**

1.  Create a new file named `cookiebot-app-policy.hcl`.
2.  Add the following content. This policy allows **only read access** to the specific path where we stored the Cookiebot secret.
    ```hcl
    # Allow read-only access to the Cookiebot application secrets
    path "secret/data/app/cookiebot" {
      capabilities = ["read"]
    }
    ```
    *(Note: We use `secret/data/...` because the KVv2 engine versions the secrets and stores the data under this sub-path).*
3.  Upload this policy to Vault:
    ```shell
    vault policy write cookiebot-app-policy @cookiebot-app-policy.hcl
    ```

> 📚 **Documentation:** [Vault Policies](https://www.google.com/search?q=https://developer.hashicorp.com/vault/concepts/policies)

-----

### **5. Creating a Token: The Application's Keycard**

Now we'll create a new token—a "keycard"—for our application. 
We will attach the policy we just created to it, giving it very limited permissions.

**Your Exercise:**

1.  Run the `token create` command, associating it with our new policy:
    ```shell
    vault token create -policy="cookiebot-app-policy"
    ```
2.  Vault will output the details of the new token. Copy the `token` value itself. 
This is the keycard our application (or Ansible, in our project) will use.

-----

### **6. Testing the Limited Token**

This is the most important step to truly understand how policies work. 
Let's use our new, less-powerful token and see what it can (and can't) do.

**Your Exercise:**

1.  Switch your terminal's identity to use the new token:
    ```shell
    export VAULT_TOKEN='hvs.YYYYYYYYYYYYYYYYYYYY' # Use the new token from step 5
    ```
2.  **Test 1 (Should Succeed):** Try to read the secret you have permission for.
    ```shell
    vault kv get secret/app/cookiebot
    ```
    This should work perfectly.
3.  **Test 2 (Should Fail):** Try to *write* to the same path.
    ```shell
    vault kv put secret/app/cookiebot telegram_token="new-value"
    ```
    This will fail with a **permission denied** error, because our policy only granted `read` capabilities.
4.  **Test 3 (Should Fail):** Try to read from a different, unauthorized path.
    ```shell
    vault kv get secret/database/cookiebot
    ```
    This will also fail with a **permission denied** error.

You have now successfully created a secure credential that can only perform one specific action. 
This is the foundation of secure automation. 
In our main project, the GitLab pipeline will automatically receive a temporary token just like this one to perform its deployment tasks safely.