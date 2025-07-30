# From Code to First Build with Java

**🎯 Lesson Objective:**

By the end of this session, you will be able to:

1.  **Explain** what CI/CD is and the core philosophy of GitLab.
2.  **Connect** the source code of a Java Spring Boot application to a GitLab project.
3.  **Register** a GitLab Runner with a `shell` executor and understand why it's our starting point.
4.  **Create and execute** your first pipeline to build the application, generating a `.jar` artifact.
5.  **Understand** the path for future lessons, which will include generating native images with GraalVM.

-----

### **Part 1: The Philosophy - Why We Automate**

Imagine building a car. You wouldn't assemble the entire vehicle and only then check if the engine starts or the brakes work. 
You test each component as it's installed.

In software, it's the same principle. 
The philosophy behind GitLab is to provide a single, unified platform for the entire software development lifecycle.

**Key Concepts (Your New Vocabulary):**

- **CI (Continuous Integration):** 
The practice of frequently merging all developers' code into a central repository. 
Each time code is pushed, an automated process kicks in to build and test it. 
This helps us find problems *early*, when they are easy and cheap to fix.

- **CD (Continuous Delivery/Deployment):** 
The next logical step. 
If all the automated tests pass, the code is automatically prepared for release (Delivery) or even deployed directly to users (Deployment).

- **Pipeline:** 
Think of it as your software assembly line.
It’s the sequence of steps (**jobs**) your code goes through, defined in a single file: `.gitlab-ci.yml`.
For example: `compile -> test -> package`.

- **Runner:** 
This is the dedicated worker that executes the jobs in your pipeline.
The GitLab server manages the workflow (the "what"), but the Runner does the actual heavy lifting (the "how").

### Gitlab University Course

Sign up on [Gitlab University](https://university.gitlab.com/) and do the Course [Getting Started with CI/CD](https://university.gitlab.com/learn/course/getting-started-with-cicd/getting-started-with-cicd/gitlab-cicd)

You can **skip** the **Hands-on Lab** from the Course.

-----

### **Part 2: Getting Your Code into GitLab**

Your GitLab instance is already running. The first step is to make it aware of your project. 
For this first lesson, we'll push the code you already have on your machine. 
This gives us a fast and reliable starting point.

1.  **Create Your Project:** In your GitLab instance, create a new blank project. Name it `cookiebot`.
2.  **Connect Your Local Code:** Open a terminal in the root directory of your Java application's source code and run the following commands.

```bash
# This command gives a nickname to your GitLab instance's URL.
git remote add origin http://gitlab.local/root/cookiebot.git

# Add all your files to be tracked by Git
git add .

# Create your first "commit" - a snapshot of your code
git commit -m "Initial commit of the Cookiebot project"

# Push your code from your machine to the GitLab server!
git push -u origin main 
# (Note: Your default branch might be 'master' instead of 'main')
```

After this, refresh your project page in GitLab. You should see all your files.

-----

### **Part 3: Hiring a Worker (Registering a `shell` Runner)**

Your GitLab project now has the blueprint, but it needs a worker to build it. Let's register one.

For this first lesson, we will use the **`shell` executor**.

**A Key Teaching Moment:** 
The `shell` executor is the simplest type. 
It runs your pipeline commands directly on the machine where the runner is installed, using the tools and environment already present on that machine. 
This is perfect for our first build because your machine is already set up with **Java and Maven**. 
In a future lesson, we will explore the `docker` executor, which provides a clean, isolated environment for every job, a more advanced and robust approach.

1. **Find Your Registration Token:** In your GitLab project, navigate to **Settings > CI/CD** and expand the **Runners** section. You will see the GitLab instance URL and a "registration token" when clicking on the 3 dots (...). Keep this page open.
2. **Start the Registration Process:** On the machine where you will run the builds (likely your own powerful development machine), open a terminal and run the `gitlab-runner register` command.
3. **The Interactive Dialogue:** The command will start a conversation. Here’s how to answer:
    - **Enter the GitLab instance URL:** Copy it from the GitLab UI.
    - **Enter the registration token:** Copy the token. It's sensitive, so handle it with care.
    - **Enter a description for the runner:** `Java Build Machine`
    - **Enter tags for the runner:** This is critical. Enter `java-build`. Tags are labels that allow us to direct specific jobs to specific, capable runners.
    - **Enter the executor:** `shell`

> 📚 **Documentation Reference:** [Registering a Runner](https://docs.gitlab.com/runner/register/)

Your runner is now registered and waiting for jobs! You can confirm this by refreshing the Runners page in GitLab.

You can run the Gitlab runner as a non-root user by using:
```shell
gitlab-runner run
```

> 📚 **Recommended Reading:** [GitLab Runner Executors](https://docs.gitlab.com/runner/executors/) (Focus on understanding the difference between `shell` and `docker`).

-----

### **Part 4: The Core Challenge - Your First Pipeline**

This is the most important part of your journey today. 
Your challenge is to create a `.gitlab-ci.yml` file that tells your new runner how to compile your Java application.

You can check examples of Gitlab CI to get inspirations:

| Gitlab CI                                                                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------|
| [Gradle](https://gitlab.com/gitlab-org/gitlab/-/blob/master/lib/gitlab/ci/templates/Gradle.gitlab-ci.yml)                                     |
| [Spring Cloud Front Deploy Demo](https://gitlab.com/gitlab-examples/spring-gitlab-cf-deploy-demo/-/blob/master/.gitlab-ci.yml?ref_type=heads) |


> 📚 **Your Most Important Resource:** [`.gitlab-ci.yml` Keyword Reference](https://docs.gitlab.com/ci/yaml/) (Bookmark this page. You will use it constantly.)

#### Your Goal:

Create a file named `.gitlab-ci.yml` in the root of your project. 
This file should define a pipeline that is automatically triggered when you `push` a change. 
The pipeline should execute the build command for your application and generate a `.jar` artifact.

**Success is not a perfect build on the first try.** 
Success is seeing your pipeline run, even if it fails. 
The logs generated by a failed job are your most valuable learning tool.

**Strategic Hints (Use these when you get stuck):**
**Where do I start?** Begin with the simplest possible pipeline to confirm everything is connected. 
This file tells GitLab to run a job named `build-job` during the `build` stage.

```yaml
# .gitlab-ci.yml

stages:
  - build
  - test

build-job:
  stage: build
  tags: # This is CRITICAL. It tells GitLab to use the runner we just made.
    - java-build
  script:
    - echo "Pipeline started! Compiling the Cookiebot..."
    - echo "I am running as user: $(whoami)"
    - echo "I am in directory: $(pwd)"
    - ls -la # List files to confirm the source code is here.
    #
    # --- YOUR REAL BUILD COMMAND GOES HERE ---
    # For a Maven project, it's usually:
    - mvn package -DskipTests
    # Using the wrapper (./mvnw) is a good practice!
  artifacts:
    paths:
      - target/*.jar
```

#### **"Command not found" error:** 
Since we are using the `shell` executor, this error means the command (`mvn`, `java`) is not available in the environment of the user running the `gitlab-runner` process. 
Make sure all your environment variables (`PATH`, `JAVA_HOME`) are correctly set for that user on the machine.

#### **How do I trigger the pipeline?**
Simply `commit` and `push` your new `.gitlab-ci.yml` file to your GitLab project. 
Then, go to the **CI/CD > Pipelines** section in the GitLab UI to watch it run in real-time.


-----

### **Extra Challenge (Once your first build runs)**

#### **Organize Your Workflow:**

- Add a job in the `test` stage that runs your application's unit tests (e.g., `mvn test`).
- What happens if the tests fail? Should the pipeline stop? (Hint: search for `when: on_success`).

Finally, take a moment to reflect. 
How did it feel to see the automation kick in? 
What was the most challenging part? 
What did the error logs teach you?

This is the core loop of CI/CD: a rapid cycle of trying, failing, learning, and improving. Welcome to the team.
