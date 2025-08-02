### **Building Native Images with Docker and GraalVM**

**🎯 Lesson Objective:**

By the end of this lesson, you will be able to:

1.  **Explain the benefits** of using the Docker executor over the shell executor.
2.  **Register a new GitLab Runner** configured to use Docker.
3.  **Modify a CI job** to run inside a specific Docker container (GraalVM).
4.  **Compile the Cookiebot** into a native executable and save it as an artifact.

-----

### **1. The "Why": Shell vs. Docker Executor**

Think of the `shell` executor as cooking in your own kitchen. 
It works, but you need to make sure you have all the right ingredients (Java, Maven) and that they are the correct versions. 
If you get a new kitchen, you have to set it all up again.

The **`docker` executor** is like being given a brand-new, pristine, pre-packaged kitchen for every single meal you cook.

- **Isolation:** Each job runs in its own clean container. Nothing from a previous job can interfere with the current one.
- **Reproducibility:** You specify the exact "kitchen" you need (e.g., `image: maven:3.8-openjdk-17`). 
Your build will work the same way today, tomorrow, or on any machine in the world that can run Docker.
- **Flexibility:** Your `build` job can use a Java container, while your `deploy` job uses an Ansible container. 
You are no longer tied to the tools installed on a single machine.

-----

### **2. Registering a Docker Runner**

It's time to "hire" a new, more specialized worker. 
We will register a new runner, but this time, we'll tell it to use Docker.

**Your Exercise:**

1.  Go to your GitLab project's **Settings > CI/CD > Runners** section to get a new registration token. It's good practice to use a new token for a new runner.
2.  On your terminal, run the registration command. You can register the runner as root to allow the usage of docker socket or as non-root user with permission to access docker.sock which allows the runner container to start other Docker containers.
    ```bash
    gitlab-runner register
    ```
3.  Follow the interactive prompts, with these key changes:
    * **Enter a description for the runner:** `My Docker Runner`
    * **Enter tags for the runner:** `docker,graalvm-build` (This is how we'll target our new job).
    * **Enter the executor:** `docker` (This is the most important step!).
    * **Enter the default Docker image:** `alpine:latest` (This is just a fallback if a job doesn't specify its own image).

You should now see two runners in your GitLab UI. We'll use tags to control which runner picks up which job.

> 📚 **Documentation:** [Docker Executor for GitLab Runner](https://docs.gitlab.com/runner/executors/docker.html)

-----

### **3. Building a Native Image with GraalVM**

GraalVM can compile Java code ahead-of-time into a native executable. 
The result is a lightning-fast startup time and significantly lower memory usage—perfect for a bot!

We will now create a new job that uses our `docker` runner and a special GraalVM container to build the native Cookiebot.

**Practical Application (The New `.gitlab-ci.yml` Job):**

Add the following job to your `.gitlab-ci.yml`. It can run in parallel with your old `build` job.

```yaml
build_native_cookiebot:
  stage: build
  tags: # We specifically target our new Docker runner
    - docker
    - graalvm-build
  image: # This is the magic! We specify the exact "kitchen" we need.
    name: ghcr.io/graalvm/native-image-community:17
    entrypoint: [""] # A technical detail to override the image's default command
  script:
    # GraalVM's native-image tool uses a lot of memory. We give it a hint.
    - export MAVEN_OPTS="-Xmx512m"
    # We use Maven to build the standard .jar file.
    #    The native-image tool needs this as input.
    - mvn package
  artifacts:
    name: "cookiebot-native-${CI_COMMIT_SHORT_SHA}"
    paths:
      - target/cookiebot # The native executable has no .jar extension
    expire_in: 1 week
```

**Your Exercise:**

1.  **Add this new job** to your `.gitlab-ci.yml`.
2.  **Commit and push** your changes.
3.  **Observe the pipeline.** You will see the `build_native_cookiebot` job start. 
Click on it and watch the log. You'll see GitLab pulling the `ghcr.io/graalvm/native-image-community:17` image before running your script.
4.  Once complete, **download the artifact** and inspect it. 
You'll have a native executable file ready for deployment!

### Extra challenge

Create a docker image with the native image executable to run Cookiebot

This process demonstrates the true power of the Docker executor: the ability to pull a highly specialized environment from a public registry, perform a complex build, and then discard the environment, leaving no trace behind. 
This is the foundation of modern, scalable CI/CD.