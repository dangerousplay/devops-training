### **Building with Speed and Intelligence**

**🎯 Lesson Objective:**

By the end of this lesson, you will be able to:

1. **Dramatically speed up** your build times using **caching**.
2. **Precisely control** when jobs run using the powerful `rules` keyword.
3. **Troubleshoot common issues** and apply best practices used by senior engineers.

-----

### **1. Caching Dependencies: Stop Re-downloading the Internet**

**The Concept:** 
Imagine you're baking a cake. 
You wouldn't run to the store for flour and sugar every single time. 
You keep them in your pantry. 
In software, Maven does the same thing: it downloads dependency files (`.jar` files for libraries) and stores them in a local pantry, located at `~/.m2/repository` on your machine.

By default, each GitLab CI job starts with an empty pantry. 
Caching tells GitLab: "Before this job starts, bring in the pantry from the last successful run. After the job finishes, save the updated pantry for next time." 
This saves an enormous amount of time.

**Practical Application (Cookiebot):**

Let's see this in action. Here is a simple build job without caching.

**Example 1: Before Caching**

```yaml
# .gitlab-ci.yml

build_cookiebot:
  stage: build
  script:
    - echo "Building Cookiebot..."
    - mvn package # Maven downloads all dependencies every single time
```

Now, let's add the `cache` keyword.

**Example 2: After Caching**

```yaml
# .gitlab-ci.yml

build_cookiebot:
  stage: build
  script:
    - echo "Building Cookiebot..."
    - mvn package
  cache:
    key: # A unique key for this cache. We tie it to the project's pom.xml file.
      files:
        - pom.xml
    paths: # The "pantry" we want to save and restore.
      - .m2/repository/
```

**Your Exercise:**

1.  Implement the `cache` configuration in your Cookiebot's `.gitlab-ci.yml`.
2.  Run the pipeline twice.
3.  Go to the job logs for both runs and compare the total execution time. You should see a significant drop on the second run because the dependencies were loaded from the cache instead of being downloaded.

> 📚 **Documentation:** [GitLab CI/CD `cache` keyword](https://docs.gitlab.com/ci/yaml/#cache)

-----

### **2. Controlling Job Execution: The Power of `rules`**

**The Concept:** You don't want every job to run all the time. 
A full deployment might only happen when you create a new version tag. 
A security scan might only be necessary when code is being merged into the `main` branch. The `rules` keyword gives you this granular control.

Here are different scenarios to show its power.

**Scenario 1: Run a Job Only on the `main` Branch**
This is useful for jobs that should only run against your primary line of code.

```yaml
deploy_to_staging:
  stage: deploy
  script:
    - echo "Deploying to our test environment..."
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"' # The job only runs if this condition is true
```

**Scenario 2: Run a Job Only for Merge Requests**
This is perfect for running code quality checks or review tools before code is merged.

```yaml
run_code_quality_scan:
  stage: test
  script:
    - echo "Running a deep code quality scan..."
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

**Scenario 3: Run a Job Only When a Git Tag is Created**
This is the standard practice for triggering a production deployment, as shown in our project diagram.

```yaml
deploy_cookiebot_to_production:
  stage: deploy
  script:
    - echo "Deploying a new version of Cookiebot!"
    - ./deploy_script.sh
  rules:
    - if: $CI_COMMIT_TAG # This condition is true only when you push a tag
```

**Your Exercise:**

* Create a new job in your pipeline called `hello_on_main`.
* Use a `rule` to make it run *only* when you push a commit to your `main` branch. Test it by pushing to a different branch (it shouldn't run) and then to `main` (it should run).

> 📚 **Documentation:** [GitLab CI/CD `rules` keyword](https://docs.gitlab.com/ci/yaml/#rules)

-----

### **3. Troubleshooting & Best Practices**

This is what separates a good engineer from a great one: understanding not just *how* to do something, but *why*, and how to fix it when it breaks.

**How to Troubleshoot a Slow Pipeline:**
Your first tool is the pipeline view in GitLab. It shows you the duration of each job.

1.  **Identify the bottleneck:** Is one job taking significantly longer than others? Click on it.
2.  **Analyze the job log:** Inside the log, which command is taking all the time? If you see Maven or Gradle downloading hundreds of lines of dependencies, you know you have a caching problem. If it's your test suite, you might need to optimize the tests themselves.

**Best Practice: `cache` vs. `artifacts`**
This is a common point of confusion, and understanding it is critical.

* **`artifacts` (The Result):**
    * **Purpose:** To pass the *output* of a job to other jobs in later stages.
    * **Example:** Your `build` job creates `cookiebot.jar`. You declare it as an artifact so the `deploy` job can access it and copy it to the server.
    * **Guarantee:** GitLab guarantees that artifacts from a successful job will be available.
* **`cache` (The Pantry):**
    * **Purpose:** To speed up a single job or multiple jobs by saving and restoring temporary files.
    * **Example:** Your `~/.m2/repository` directory. The `deploy` job doesn't need it, but the *next* `build` job does.
    * **Guarantee:** GitLab does *not* guarantee the cache will be available. It's a best-effort optimization. Your jobs should still work (though slower) if the cache is empty.

**Best Practice: Document Your Environment**
Even though you're using a shell runner, the tools on that machine (Java, Maven) can change. A great practice is to add a "setup" job at the beginning of your pipeline that prints the versions of your core tools.

```yaml
check_environment:
  stage: .pre # A special stage that always runs first
  script:
    - echo "Checking build environment..."
    - java -version
    - mvn -v
```

This log becomes invaluable for debugging six months from now when something mysteriously breaks after a system update.

**Your Final Task for this Lesson:**

* Review your Cookiebot `.gitlab-ci.yml`.
* Can you clearly explain the purpose of every line?
* Can you identify where you are using (or should be using) `artifacts` versus `cache`?

Mastering these concepts will give you the confidence to not only build pipelines but to build them well, making you an incredibly valuable asset to any DevOps team.