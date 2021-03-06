+++
title = "Snyk Security"
chapter = false
weight = 1
+++

## Snyk Security

### Expected Outcome
- Install Snyk
- Test open source dependencies of the Petstore application, have it fail, remediate security issue, and then have it pass.
- Test a docker image for security vulnerabilities

**Lab Requirements:** Automation lab must be completed

**Average Lab Time:** 30 minutes

### Introduction
This module is designed to introduce scanning **open source dependencies** of the application and the **Docker** container that is created during this workshop. The files required for this module have already been created and reside in the /modules/snyk folder. You will copy these over while following the steps below.

### Background
**Snyk** is a **SaaS** offering that organizations use to **find, fix, prevent and monitor** open source dependencies. Snyk is a developer first platform that can be easily integrated into the Software Development Lifecycle (SDLC).

At this point of the module, the `Petstore` application is created, so we will look to insert **Snyk** as part of an important security gate during the build process.

This module will demonstrate how to fail a build when high severity issues are found so that remediation can take place.

### Snyk CLI
The Snyk command line interface (CLI) has three key commands for this exercise:
- `snyk auth` which links the CLI to your account and authorizes it to perform tests.
  - We will utilize Amazon's System Manager Parameters to store this token to avoid hard-coding tokens.
  - Alternatively to using **snyk auth**, you can also set an **environment variable**  `SNYK_TOKEN` which the CLI will automatically detect.
- `snyk test` performs the actual test and can fail a build
- `snyk monitor` posts a snapshot for continuous monitoring and reporting on the `snyk.io` interface where you created your account.

### Tasks for this Lab
For the purposes of this module, **Snyk** will be inserted into two key processes, when the **application** is being built, and when the **Docker container** is created.

This module has five sections:
1. Obtain a token for testing from `https://snyk.io/`
2. Setting up application scanning
3. Setting up Docker analysis
4. Running a test and fixing the issue(s) detected
5. Viewing the status on the Snyk dashboard

### Create a Snyk Account and Obtain a Token
Obtain an account and setting up the credentials for this exercise:

- You will sign up to https://app.snyk.io/signup using `Google`, `Bitbucket` or `Github` credentials. **Snyk** utilizes these services for authentication and does not store passwords.

![Snyk login](/images/snyk_1_login.png)


- Once signed up you will navigate to **your name** (top right), and select **Account Settings**

![Snyk account settings](/images/snyk_2_account_settings.png)

- Under **API Token**, click **Show** and copy this value, it will be unique for each user.

![Snyk API token](/images/snyk_3_api_token.png)

- Clicking **Show** reveals the token to copy.

![Snyk API token show](/images/snyk_3b_api_token_show.png)

### Save Password to Session Manager
Run the following command, replacing `abc123`, with **your unique token**. This places the token in the session parameter manager.
```
aws ssm put-parameter --name "snykAuthToken" --value "abc123" --type SecureString
```

### Setup the Application Scanning
We want to insert testing with **Snyk** after `maven` has built the application. The simplest method is to insert commands to download, authorize and run the **Snyk** commands after Mvn has built the application/dependency tree.

In `modules/snyk/Dockerfile`, we have inserted the following commands to perform these actions

Set an environment variable from a value passed to the `docker build` command, this will contain the token for **Snyk**. By using an environment variable, **Snyk** will automatically detect the token when used.

```
#~~~~~~~SNYK Variable~~~~~~~~~~~~
# Declare Snyktoken as a build-arg
ARG snyk_auth_token
# Set the SNYK_TOKEN environment variable
ENV SNYK_TOKEN=${snyk_auth_token}
#~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

Download **Snyk**, run a test, looking for medium to high severity issues, and if the build succeeds, post the results to **Snyk** for monitoring and reporting. If a new vulnerability is found, you will be notified.

```
# package the application
RUN mvn package -Dmaven.test.skip=true

#~~~~~~~SNYK test~~~~~~~~~~~~
# download, configure and run snyk. Break build if vulns present, post results to `https://snyk.io/`
RUN curl -Lo ./snyk "https://github.com/snyk/snyk/releases/download/v1.210.0/snyk-linux"
RUN chmod -R +x ./snyk
#Auth set through environment variable
RUN ./snyk test --severity-threshold=medium
RUN ./snyk monitor
```

### Setting Up Docker Scanning
Later in the build process, a docker image is created. We want to analyze it for vulnerabilities. We will do this in `buildspec.yml`. First, pull the **Snyk** token `snykAuthToken` from the `parameter store`:
```
env:
  parameter-store:
    SNYK_AUTH_TOKEN: "snykAuthToken"
```

In the `prebuild` phase, we will install Snyk
```
phases:
  pre_build:
    commands:

      - PWDUTILS=$(pwd)
      - curl -Lo ./snyk "https://github.com/snyk/snyk/releases/download/v1.210.0/snyk-linux"
      - chmod -R +x ./snyk
```

In the `build` phase we will pass the token to the **docker compose** command where it will be retrieved in the Dockerfile code we previously setup to test the application:
```
build:
    commands:
      - docker build --build-arg snyk_auth_token=$SNYK_AUTH_TOKEN -t $REPOSITORY_URI:latest .
```

Next we will **authorize** the **Snyk** instance for testing the Docker image that’s produced. If it passes we will pass the results to **Snyk** for monitoring and reporting.
```
build:
    commands:
      - $PWDUTILS/snyk auth $SNYK_AUTH_TOKEN
      - $PWDUTILS/snyk test --docker $REPOSITORY_URI:latest
      - $PWDUTILS/snyk monitor --docker $REPOSITORY_URI:latest
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
```

In terminal, **navigate** to this folder:
```
cd ~/environment/aws-modernization-workshop/
```

To try this module, let us copy the **Snyk** versions over to our build:
```
cp modules/snyk/Dockerfile modules/containerize-application/Dockerfile
cp modules/snyk/buildspec.yml buildspec.yml
```

### Exercise - Testing
In the `Containerize Application` lab you saw how to build your application. In this exercise you will try to run your build, which will fail due to security vulnerabilities being found. While normally done during the code development phase, we will take you through the process of fixing the vulnerability, and then re-running the exercise to see the build succeed.

*Save changes*:
```
git commit -am "snyk"
```

*Push*:
```
git push -f codecommit master
```

Now in `CodeBuild`, look at your **build history**. Note it may take a minute or two for the new scan to run.

![Snyk build](/images/snyk_4_Build.png)

Let’s look at why this failed. We see security vulnerabilities were found and we’re told **how** to fix it!

```
Testing /usr/src/app...
✗ Medium severity vulnerability found in org.primefaces:primefaces
Description: Cross-site Scripting (XSS)
Info: https://snyk.io/vuln/SNYK-JAVA-ORGPRIMEFACES-31642
Introduced through: org.primefaces:primefaces@6.1
From: org.primefaces:primefaces@6.1
Remediation:
Upgrade direct dependency org.primefaces:primefaces@6.1 to org.primefaces:primefaces@6.2 (triggers upgrades to org.primefaces:primefaces@6.2)
✗ Medium severity vulnerability found in org.primefaces:primefaces
Description: Cross-site Scripting (XSS)
Info: https://snyk.io/vuln/SNYK-JAVA-ORGPRIMEFACES-31643
Introduced through: org.primefaces:primefaces@6.1
From: org.primefaces:primefaces@6.1
Remediation:
Upgrade direct dependency org.primefaces:primefaces@6.1 to org.primefaces:primefaces@6.2 (triggers upgrades to org.primefaces:primefaces@6.2)
Organisation: sample-integrations
Package manager: maven
Target file: pom.xml
Open source: no
Project path: /usr/src/app
Tested 37 dependencies for known vulnerabilities, found 2 vulnerabilities, 2 vulnerable paths.
The command '/bin/sh -c ./snyk test' returned a non-zero code: 1
[Container] 2018/11/09 03:46:22 Command did not exit successfully docker build --build-arg snyk_auth_token=$SNYK_AUTH_TOKEN -t $REPOSITORY_URI:latest . exit status 1
[Container] 2018/11/09 03:46:22 Phase complete: BUILD Success: false
[Container] 2018/11/09 03:46:22 Phase context status code: COMMAND_EXECUTION_ERROR Message: Error while executing command: docker build --build-arg snyk_auth_token=$SNYK_AUTH_TOKEN -t $REPOSITORY_URI:latest .. Reason: exit status 1
```

### Exercise - Fixing the Vulnerability
According to the remediation, we need to fix the `PrimeFaces` dependency and update it from version `6.1` to `6.2`.

Let us pretend the developer fixed it and checked it in, coming back into the pipeline. This is done by changing.

`~/environment/aws-modernization-workshop/modules/containerize-application/app/pom.xml`

*Changing:*
```
<version.primefaces>6.1</version.primefaces>
```

*To:*
```
<version.primefaces>6.2</version.primefaces>
```

Run this command to copy over our fixed version in the lab:
```
cp modules/snyk/pom.xml modules/containerize-application/app/
```

*Save changes:*
```
git commit -am "Fix vulnerable open source dep."
```

*Push:*
```
git push -f codecommit master
```

This time check `Code Builder` and we see it succeeded.

![Snyk build](/images/snyk_4b_Build.png)

```
Tested 37 dependencies for known vulnerabilities, no vulnerable paths found.
Next steps:
- Run `snyk monitor` to be notified about new related vulnerabilities.
```

The vulnerability is fixed and the build succeeded!

Next, we also see **Snyk** successfully scanned the `Docker Image` and there were no package dependency issues with our Docker container!

```
Container] 2018/11/09 03:54:14 Running command $PWDUTILS/snyk test --docker $REPOSITORY_URI:latest
Testing 300326902600.dkr.ecr.us-west-2.amazonaws.com/petstore_frontend:latest...
Organisation: sample-integrations
Package manager: rpm
Docker image: 300326902600.dkr.ecr.us-west-2.amazonaws.com/petstore_frontend:latest
✓ Tested 190 dependencies for known vulnerabilities, no vulnerable paths found.
```

### Viewing Reporting
* Navigate back to `https://snyk.io/`.
* You will see your  Docker Image and Java application displayed
* Click **View Report**
* Set frequency project will be checked for vulnerabilities with the drop down list
* Click on **View Report** and then the **Dependencies** tab to see what libraries were used. Click **View All Dependencies**
* Use the **Integrations** tab (optionally) to connect and automate creation of fixes against a **code repository**.

![Snyk UI](/images/snyk_5_snykUI.png)
