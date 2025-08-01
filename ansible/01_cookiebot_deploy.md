### **Cookiebot deploy**

**🎯 Lesson Objective:**

By the end of this lesson, you will be able to write an Ansible playbook that:

1.  **Copies** the Cookiebot native executable, its properties file, and SSL certificates to a target Linux server.
2.  **Creates and manages a `systemd` service** to ensure Cookiebot runs as a proper background process.
3.  **Intelligently restarts** the service only when necessary using **handlers**.
4.  **Organize your Ansible project** in a clean, professional directory structure.

-----

### **1. The "Why": From Manual Steps to a Reliable Playbook**

Why not just write a shell script? Ansible gives us superpowers that scripts don't:

* **Idempotency:** You can run an Ansible playbook 100 times, and it will only make changes the first time (or if something has drifted from the desired state). 
It won't keep creating the same user or copying the same file if it's already correct.
* **State Declaration:** You describe the *end state* you want ("this file should exist," "this service should be running"), and Ansible figures out *how* to get there.
* **Modularity & Readability:** Playbooks are easy to read and manage, breaking down complex tasks into logical plays and tasks.

-----

### **2. Key Concepts & Modules for Deployment**

We'll use a few essential Ansible modules to accomplish our goal:

* **`copy`:** For simple file transfers. Perfect for copying the pre-built native executable and the SSL certificates, which don't need to be modified.
* **`template`:** The "smart copy." It uses the Jinja2 templating engine to transfer a file while injecting variables. 
This is ideal for configuration files, like our `systemd` service unit, where we might want to define the username or working directory dynamically.
* **`systemd`:** The module for managing services on modern Linux systems. We'll use it to enable and start our Cookiebot service.
* **Handlers:** These are special tasks that only run when "notified" by another task. This is the professional way to handle service restarts. 
Instead of restarting the service every time the playbook runs, we'll only restart it *if* the executable or its configuration has actually changed.

-----

### **3. Professional Project Structure**

Organizing your files makes your automation reusable and easy to understand. 
Let's structure our project like this:

```
ansible/
├── playbook.yml          # Our main playbook file
├── inventory.ini         # Defines the server(s) we will manage
├── files/                # For static files to be copied
│   └── certs/
│       ├── cookiebot.crt
│       └── cookiebot.key
└── templates/            # For template files with variables
    └── cookiebot.service.j2
```

-----

### **4. Building the Playbook, Step-by-Step**

Let's create our `playbook.yml`.

**Step 1: The Basic Structure and Copying Files**
First, we define which hosts to target and create tasks to copy our files over.

```yaml
# playbook.yml
---
- name: Deploy and Configure Cookiebot
  hosts: cookiebot_server
  become: yes # Most of these tasks require root privileges (sudo)

  tasks:
    - name: Create a dedicated directory for the bot
      file:
        path: /opt/cookiebot
        state: directory
        owner: cookiebot_user # Assumes a user 'cookiebot_user' exists
        group: cookiebot_user
        mode: '0755'

    - name: Copy the new Cookiebot native executable
      copy:
        src: ../target/cookiebot # Assuming the executable is in the parent 'target' dir
        dest: /opt/cookiebot/cookiebot
        owner: cookiebot_user
        group: cookiebot_user
        mode: '0755' # Make it executable
      notify: Restart Cookiebot # We'll define this handler later

    - name: Copy SSL certificates
      copy:
        src: files/certs/
        dest: /opt/cookiebot/certs/
        owner: cookiebot_user
        group: cookiebot_user
        mode: '0644'
      notify: Restart Cookiebot
```

* `become: yes` tells Ansible to use `sudo` for the tasks.
* The `notify` keyword is a hook. It says, "If this task makes a change, tell the handler named 'Restart Cookiebot' to run at the end of the playbook."

**Step 2: Creating the `systemd` Service with a Template**
We need a service file to tell Linux how to run our bot.

Now, add the task to your `playbook.yml` to use this template:

```yaml
# Add this task to your playbook.yml
    - name: Create systemd service file for Cookiebot
      template:
        src: templates/cookiebot.service.j2
        dest: /etc/systemd/system/cookiebot.service
        owner: root
        group: root
        mode: '0644'
      notify: Restart Cookiebot
```

**Step 3: Defining Handlers and Managing the Service**
Finally, we define the `handlers` section at the end of our playbook. 
This section contains the "Restart Cookiebot" task that our other tasks have been notifying.

```yaml
# Add this section at the end of playbook.yml, at the same indentation level as 'hosts' and 'tasks'
  handlers:
    - name: Restart Cookiebot
      systemd:
        name: cookiebot
        state: restarted
        enabled: yes # Ensures the service starts on boot
        daemon_reload: yes # Reloads systemd to recognize the new service file
```

This handler will only run ONCE at the end of the play, even if multiple tasks notify it. 
And it will only run if at least one of those tasks reported a "changed" status. 
This is peak efficiency!

-----

### **Your Exercise**

1.  **Create the project structure** as shown above.
2.  **Create the `inventory.ini` file** with the IP address of your target VM.
3.  **Create the `cookiebot.service.j2` template file.**
4.  **Assemble the complete `playbook.yml`** with the tasks and handlers.
5.  **Run the playbook** from your local machine against a VM or container:
    `ansible-playbook -i inventory.ini playbook.yml`
6.  **Verify:** SSH into your server and check the service status with `systemctl status cookiebot`.
7.  **Test Idempotency:** Run the playbook a second time. Observe the output. It should report "ok" for most tasks and "changed=0", and the handler should not run.
