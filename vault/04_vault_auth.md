### **4 - Vault Authentication Methods in Practice**

**🎯 Lesson Objective:**

By completing this lesson, you will be able to:

1.  **Enable, configure, and use** the `userpass` auth method for human operators.
2.  **Enable and configure** the `JWT/OIDC` auth method for machine-to-machine authentication.
3.  **Create a Vault Role** that securely binds a GitLab CI job to a set of Vault policies.
4.  **Understand and troubleshoot** the JWT authentication flow within a GitLab CI pipeline.

-----

### Prerequisites

- Read [Vault Authentication](https://developer.hashicorp.com/vault/docs/concepts/auth)
- Read [Vault Userpass auth method](https://developer.hashicorp.com/vault/docs/auth/userpass)

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

### **Part 2: Human Authentication with `userpass`**

**The Concept:** The `userpass` auth method is the most straightforward way to handle human authentication. 
It allows users to log in with a simple username and password that are stored directly inside Vault. 
This is perfect for operators, developers, or anyone who needs to interact with Vault via the command line or the UI.

#### **Exercise 1: Enabling and Creating a User**

1.  **Command:** Enable the `userpass` auth method at its default path.
    ```shell
    vault auth enable userpass
    ```
2.  **Action:** Let's create a policy specifically for a human operator who needs to manage the Cookiebot application's secrets. Create a file named `cookiebot-admin-policy.hcl`:
    ```hcl
    # Grant full control over all secrets for the Cookiebot app
    path "secret/data/cookiebot/*" {
      capabilities = ["create", "read", "update", "delete", "list"]
    }
    ```
    Now, write it to Vault:
    ```shell
    vault policy write cookiebot-admin cookiebot-admin-policy.hcl
    ```
3.  **Command:** Create a new user named `melany` with a password and attach the `cookiebot-admin` policy to her account.
    ```shell
    vault write auth/userpass/users/melany \
        password="super-secure-password-123" \
        policies="default,cookiebot-admin"
    ```

>  **❓ Question:** The documentation mentions you can mount auth methods at custom paths (e.g., `vault auth enable -path=team-a-logins userpass`). What is a practical reason a company might do this?

#### **Exercise 2: Logging In as a User**

Now, let's experience logging in as the user we just created.

1.  **Action:** First, log out of your root token session to simulate being a new user.
    ```shell
    vault logout
    # You can also just unset the VAULT_TOKEN variable
    # unset VAULT_TOKEN
    ```
2.  **Command:** Log in using the `userpass` method.
    ```shell
    vault login -method=userpass username=melany
    ```
3.  **Action:** Vault will prompt you for your password. Enter `super-secure-password-123`.
4.  **Observation:** Upon success, Vault provides a new token. Look at the `token_policies` field in the output. It should list `default` and `cookiebot-admin`.
5.  **Command:** Test your permissions. This command should succeed because of the `cookiebot-admin` policy.
    ```shell
    vault kv put secret/cookiebot/test-from-melany value=success
    ```

>  **❓ Question:** What is the "lease" or `token_duration` of the new token you received? What will happen when this duration expires?

-----

### **Part 3: Deep Dive into GitLab JWT Authentication**

Understanding *how* the JWT auth method works is key to troubleshooting it. 
It's like being able to read the information on a keycard to understand why a door won't open.

#### **What is a JWT? (A Quick Analogy)**

Think of a JWT (JSON Web Token) as a secure, self-contained digital ID card.

* **Issuer:** It's issued by a trusted authority (in our case, **GitLab**).
* **Claims:** It contains specific pieces of information, called "claims" (like your name, ID number, and expiration date). For GitLab, these claims are things like `project_id`, `user_login`, and `ref` (the branch or tag name).
* **Signature:** It's digitally signed by the issuer. This signature proves that the information on the card is authentic and hasn't been tampered with.

When our GitLab job wants to log into Vault, it doesn't present a password. It presents this temporary, signed ID card (the JWT). Vault's job is to check the signature to make sure it's really from GitLab and then read the claims on the card to see if this job matches the rules we defined in our Vault Role.

-----

#### **How to Configure `.gitlab-ci.yml` to Securely Generate and Use a JWT**

To securely generate a JWT for a specific service like Vault, we **must** use the `id_tokens` keyword in our `.gitlab-ci.yml`. This is the modern and recommended approach.

**Why `id_tokens`? The "Audience" Claim (`aud`)**

The `id_tokens` keyword lets us create a JWT for a specific **audience**. Think of it this way: your driver's license is a general ID, but your company access badge is a specific ID valid *only* for entering your office building.

The `aud` (audience) claim is like writing "For Vault Use Only" on our digital ID card. It adds a crucial layer of security:

1.  We configure GitLab to issue a token intended only for our Vault server.
2.  We configure Vault to **only accept tokens intended for it**.

This prevents a token generated for Vault from being accidentally or maliciously used to authenticate against another service (like AWS, GCP, etc.).

**Practical Application: The Login Script**

Here is the definitive way to configure your deploy job to authenticate with Vault.

```yaml
# In your .gitlab-ci.yml

deploy_to_production:
  stage: deploy
  # 1. Define the ID token we need.
  id_tokens:
    VAULT_ID_TOKEN: # We choose this environment variable name.
      aud: 'https://vault.service.local' # The audience must match Vault's configuration.
  
  rules:
    - if: $CI_COMMIT_TAG
  
  script:
    - echo "Authenticating to Vault with a specific ID token..."
    - export VAULT_ADDR=http://localhost:8200
    
    # 2. The JWT is now available in the $VAULT_ID_TOKEN variable.
    #    We use it to log in to the 'jwt' auth path and the specific role.
    - export VAULT_TOKEN=$(vault write -field=token auth/jwt/login role="cookiebot-deploy-role" jwt="$VAULT_ID_TOKEN")
    
    # 3. If successful, any subsequent commands in this job are now authenticated with Vault.
    - echo "Authentication successful."
    - ansible-playbook -i ... # Ansible can now run and fetch secrets.
```

> 📚 **Documentation:** [GitLab ID Token Authentication](https://docs.gitlab.com/ci/secrets/id_token_authentication/)

-----

#### **Exercise 6: Decoding and Inspecting the GitLab ID Token**

Let's see the audience claim in action.

**1. Modify `.gitlab-ci.yml`**
Add a temporary debugging job to your pipeline.

```yaml
debug_jwt_job:
  stage: test
  id_tokens:
    MY_DEBUG_TOKEN:
      aud: 'my-test-audience'
  script:
    - echo "--- Decoding a specific ID Token for debugging ---"
    - echo "WARNING: Do NOT do this in a production environment!"
    - echo $MY_DEBUG_TOKEN
    - echo "--------------------------------------------"
```

**2. Run the Pipeline and Decode the Token**
Commit this change, run the pipeline, and copy the long token string from the `debug_jwt_job` log. Paste it into **[jwt.io](https://jwt.io/)**.

* **Observation:** Look at the **PAYLOAD** data. In addition to `project_id`, `ref`, etc., you will now see an `aud` claim with the value `my-test-audience`. This proves that GitLab correctly generated the token for your specified audience.

> **❓ Question:** How does creating a token with a specific `aud` claim prevent a compromised token from being used against other cloud services that might also be configured to trust your GitLab instance?

-----

#### **Updating the Vault Role to Expect an Audience**

For this to work, we must update our Vault role to check for the correct audience.

**Exercise 7: Making the Vault Role Audience-Aware**

1.  **Action:** Make sure you are logged in with your **root token**.
2.  **Command:** We will update the `cookiebot-deploy-role` we created in the previous lesson by adding the `bound_audiences` parameter. It must exactly match the `aud` value from your `.gitlab-ci.yml`.
    ```shell
    vault write auth/jwt/role/cookiebot-deploy-role \
        role_type="jwt" \
        policies="cookiebot-deploy-policy" \
        token_ttl="5m" \
        user_claim="user_login" \
        bound_claims_type="glob" \
        bound_audiences="https://vault.service.local" \
        bound_claims='{
            "project_id": "12345",
            "ref_type": "tag"
        }'
    ```
    *(Note: We've also made the `bound_claims` more specific to tags now).*

-----

#### **Troubleshooting the Pipeline**

With this new configuration, you have a new, common point of failure to check.

* **Error: "claim 'aud' does not match any of the allowed audiences"**
    * **Meaning:** This is a clear message. The JWT was presented, but the `aud` claim inside it (e.g., `my-app`) does not match any of the `bound_audiences` configured in the Vault role (e.g., `https://vault.service.local`).
    * **Solution:**
        1.  Check the `aud` value you have set in your `.gitlab-ci.yml` under the `id_tokens` keyword.
        2.  Check your Vault role configuration with `vault read auth/jwt/role/cookiebot-deploy-role`.
        3.  Ensure the `aud` from your YAML file **exactly matches** one of the values in the `bound_audiences` list in Vault. They are case-sensitive\!