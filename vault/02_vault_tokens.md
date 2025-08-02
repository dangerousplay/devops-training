### **2 - Understanding the Lifecycle of a Vault Token**

**🎯 Lesson Objective:**

By completing the following exercises, you will be able to:

1.  **Inspect** the metadata of different types of tokens.
2.  **Create** tokens with specific time-to-live (TTL) properties.
3.  **Renew and Revoke** tokens using both the token and its accessor.
4.  **Explain** the key differences between token types and their use cases.

-----

### **Prerequisites**

Read the official Hashicorp documentation [Introduction to tokens](https://developer.hashicorp.com/vault/tutorials/get-started/introduction-tokens)

### **Setup: Preparing Your Lab Environment**

For all exercises, you will need a running Vault dev server.

1.  Open your terminal and start the server:
    ```shell
    vault server -dev
    ```
2.  Copy the **Root Token** that is printed in the output.
3.  Open a **new terminal window** and set up your environment variables:
    ```shell
    export VAULT_ADDR='http://127.0.0.1:8200'
    export VAULT_TOKEN='<PASTE_THE_ROOT_TOKEN_HERE>'
    ```
4.  Confirm your connection with command `vault status`.

-----

### **Lab Exercises**

#### **Exercise 1: Inspecting the Root Token**

The root token is the first and most powerful token. Let's look at its properties.

1.  **Command:** Look up your current token (the root token).
    ```bash
    vault token lookup
    ```
2.  **Observation:** Examine the output fields, specifically `token_duration`, `token_renewable`, and `policies`.

> **❓Question:** Based on the documentation you read, why is the `token_duration` for a root token infinite (`∞` or `0s` TTL) and why is it not renewable?

-----

#### **Exercise 2: Standard Service Tokens and Accessors**

Now let's create a more standard token, like one an application would use, and explore the difference between a token and its accessor.

1.  **Command:** Create a new token with the `default` policy.
    ```shell
    vault token create -policy="default"
    ```
2.  **Observation:** The command returns a new `token` and a `token_accessor`. Save both of these values.
3.  **Command:** Use the new token's accessor to look up its properties.
    ```shell
    vault token lookup -accessor <PASTE_THE_NEW_ACCESSOR_HERE>
    ```
4.  **Observation:** Notice the details you can see: `creation_ttl`, `policies`, `renewable`, etc.

> **❓Question:** According to the documentation, what is the primary purpose of an accessor? Why is it safer to log or monitor an accessor than the token itself?

-----

#### **Exercise 3: Managing Time-to-Live (TTL) and Renewals**

Tokens should not live forever. Let's experiment with their lifecycle.

1.  **Command:** Create a very short-lived token that is valid for 60 seconds (`1m`) but has a maximum lifetime of 180 seconds (`3m`).
    ```shell
    vault token create -policy="default" -ttl="1m" -explicit-max-ttl="3m"
    ```
2.  **Observation:** Save the new token value. We'll need it for the next steps.
3.  **Command:** Immediately look up the token to see its initial TTL.
    ```shell
    vault token lookup <PASTE_THE_SHORT_LIVED_TOKEN>
    ```
4.  **Command:** Now, wait for about 30 seconds and run the lookup command again.
    ```shell
    sleep 30
    vault token lookup <PASTE_THE_SHORT_LIVED_TOKEN>
    ```
5.  **Observation:** Notice that the `ttl` value has decreased. The clock is ticking\!
6.  **Command:** Before the TTL expires, renew the token.
    ```shell
    vault token renew <PASTE_THE_SHORT_LIVED_TOKEN>
    ```
7.  **Observation:** Look at the `token_duration`. It should be reset back to its original value (`1m`).

> **❓Question:** Explain in your own words the difference between `ttl` and `explicit-max-ttl` based on what you've observed. What do you predict will happen if you try to renew the token after 3 minutes have passed?

-----

#### **Exercise 4: Revoking a Token**

Sometimes, a token might be leaked or is no longer needed. We must be able to revoke it immediately.

1.  **Command:** Create a new token and save its **accessor**.
    ```shell
    vault token create -policy="default"
    ```
2.  **Command:** Revoke the token using its **accessor**. Using the accessor is a best practice because it doesn't require you to handle the secret token itself.
    ```shell
    vault token revoke -accessor <PASTE_THE_ACCESSOR_FROM_STEP_1>
    ```
3.  **Observation:** The command should return a success message.
4.  **Command:** Now, try to look up the token you just revoked (using the token value itself, if you saved it).
    ```shell
    vault token lookup <PASTE_THE_REVOKED_TOKEN>
    ```
5.  **Observation:** You will receive an error. The token is invalid and no longer exists in Vault.

> **❓Question:** Describe a real-world scenario from the documentation (or one you can imagine) where you would need to revoke a token immediately.

-----

#### **Exercise 5: Exploring Special Token Types (Advanced)**

The documentation mentions orphan and periodic tokens. Let's create them to see how they differ.

1.  **Command:** Create an **orphan** token.
    ```shell
    vault token create -orphan -policy="default"
    ```
2.  **Observation:** Look up this token. Note the `orphan: true` property.
3.  **Command:** Create a **periodic** token that is valid for 1 hour.
    ```shell
    vault token create -period="1h" -policy="default"
    ```
4.  **Observation:** Look up this token. Note the `period` is set, and there is no `explicit_max_ttl`. It can be renewed forever as long as it's renewed within its one-hour period.


> **❓Question:** Based on the documentation, what is the key benefit of an orphan token? For what type of workload would a periodic token be a better choice than a standard token with a `max_ttl`?