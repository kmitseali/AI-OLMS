# SET 3 Software Engineering Lab Internal II Solutions

**Original paper:** `Set-3.docx` in the MORNING folder.  
**Format:** Question → One-line explanation → Solution → Expected result; scenario questions receive one-line answers.

## Quick navigation

- [Before starting](#before-starting)
- [Q1 Flask with Docker Compose](#q1-flask-with-docker-compose)
- [Q2 Jenkins Scripted Pipeline](#q2-jenkins-scripted-pipeline)
- [Q3 Email notifications and GitHub webhooks](#q3-email-notifications-and-github-webhooks)
- [Q4 Car application on AWS EC2](#q4-car-application-on-aws-ec2)
- [Verification and sources](#verification-and-sources)

## Before starting

- **Local commands:** Windows PowerShell. **EC2 commands:** Ubuntu SSH terminal. Code blocks labelled YAML, Dockerfile, Java, Python, XML or Groovy are file contents unless explicitly described otherwise.
- Start Docker Desktop with its Linux engine. Check `docker version`, `docker compose version`, `git --version`, `java -version`, `mvn -version`, `kubectl version --client`, and `minikube version` as required by the question. Install Minikube using its [official guide](https://minikube.sigs.k8s.io/docs/start/) if missing, then reopen PowerShell.
- Keep practice files under `$HOME\se-lab-work\set3`; this answer file stays directly in the MORNING folder. Use a separate subfolder for each question.
- Jenkins is assumed to be installed and running: open [Jenkins](http://localhost:8080) and sign in with your existing account. Avoid mapping another local container to host port 8080 while Jenkins uses it; a question that explicitly requires 8080 explains how to handle the conflict.
- Replace placeholders such as `PUBLIC_IP`, `YOUR_EMAIL`, and `YOUR_NGROK_DOMAIN`. Use spaces in YAML, not tabs. Enable **File Explorer → View → Show → File name extensions** to avoid `.yml.txt` files.

### Repository checked for this paper

[Open the original GitHub repository](https://github.com/anujyog1/carrepo). Fresh read-only inspection on **20 November 2026** found:

| Item | Verified value |
|---|---|
| Branch | `master` |
| Inspected commit | `16e00bc2ee065277eb71a394210b5b51437aac05` |
| Maven project root | Repository root, beside `pom.xml` |
| Build output | `target/bookcar.war` |
| Source tests | None in `src/test` |

**Important:** JSP-only WAR; no Java unit tests are included. `Menu.jsp` has an uppercase M. This is not a Spring Boot executable JAR.

Repository inspection did not change GitHub. Webhook demonstrations below require a **lab repository you own or are authorized to administer**; use your own fork/copy and its URL rather than editing the question paper’s repository.

---

## Q1 Flask with Docker Compose

### Question

> Q1. You are appointed as a DevOps engineer in a software development company. The development team wants to deploy a simple Python Flask application using Docker Compose. The application should run inside a Docker container, and developers should be able to modify the application code and rebuild it easily during development. [20 Marks]

**One-line explanation:** Build and run a Flask container through Compose, then rebuild it after changing the application code.

### Solution

#### Step 1 Create the local working folder

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set3\q1" | Out-Null
Set-Location "$HOME\se-lab-work\set3\q1"
```

Create each named file in this folder using Notepad or VS Code, paste its code, and save it. In Notepad use **Save as type → All files** and keep the exact extension. File contents below are not terminal commands.

#### Step 2 Write app.py

```python
import os
from flask import Flask
app = Flask(__name__)

@app.get("/")
def home():
    return os.getenv("MESSAGE", "Hello from Flask + Docker!")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

The message is environment-configurable so the same image can serve different messages in the two-application scenario. `0.0.0.0` makes Flask reachable outside its container.

**File: `requirements.txt`**

```text
Flask==3.1.2
```

**File: `Dockerfile`**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```

#### Step 3 Write docker-compose.yml

```yaml
name: set3-flask
services:
  web:
    build: .
    ports:
      - "5000:5000"
```

#### Step 4 Build and open the app

```powershell
docker compose config --quiet
docker compose up -d --build
docker compose ps
docker compose logs --tail 30 web
```

Open **`http://localhost:5000`** and verify **Hello from Flask + Docker!**. If port 5000 is occupied, stop the conflicting app or change the host side of the mapping.

#### Step 5 Modify and rebuild

Change the default message inside `app.py`, save it, then run:

```powershell
docker compose up -d --build web
```

Refresh the browser. `--build` rebuilds the copied Python source; a simple `restart` would reuse the old image. View logs for Python syntax errors if the container exits. Stop/remove the stack with `docker compose down`.

**Expected result:** Flask runs inside Docker, is reachable through the mapped port, and shows the new message after a rebuild. Scenario answers below give the requested modifications directly without a second execution walkthrough.

### Scenario-based questions and one-line answers

1. **A developer has deployed a Flask application displaying "Hello from Flask + Docker!". The developer wants to change the message to "Hello Docker Compose!". Modify the Python code to display the updated message.                                     [ 1 Mark ]**  
   Change the route’s default return value to `"Hello Docker Compose!"` (or set `MESSAGE` to that value in the supplied environment-configurable app).

2. **A developer wants to access the Flask application through port 8080 on the host machine instead of port 5000. Modify the Docker Compose configuration to achieve this.     [ 1 Mark ]**  
   Change the web service mapping to `ports: ["8080:5000"]`; free host port 8080 first if local Jenkins uses it.

3. **A software company wants to run two Flask applications simultaneously using Docker Compose. The first application should display "Welcome to Application 1", and the second should display "Welcome to Application 2". Task: Write a Docker Compose configuration to build and run two Flask services using the same Dockerfile but different host ports. [ 3 Marks]**  
   Use `services: {app1: {build: ., environment: {MESSAGE: "Welcome to Application 1"}, ports: ["5001:5000"]}, app2: {build: ., environment: {MESSAGE: "Welcome to Application 2"}, ports: ["5002:5000"]}}` with the shared Dockerfile and environment-configurable `app.py` above.

---

## Q2 Jenkins Scripted Pipeline

### Question

> Q2. A company needs working scripted pipeline (Groovy-based Jenkins file) that automates build and test stages successfully. Demonstrate this for project from GitHub with link https://github.com/anujyog1/carrepo.git. [25 Marks]

**One-line explanation:** Use a Scripted Jenkins Pipeline to retrieve the car project, clean, compile, test, package and archive its WAR.

### Solution

#### Configure Jenkins tools once

1. **Manage Jenkins → Plugins → Available plugins:** install missing **Git**, **Maven Integration**, **Pipeline**, **Pipeline: Stage View**, **JUnit**, and **GitHub** plugins. Add **Build Pipeline** for Freestyle Pipeline View questions and **Email Extension** for email questions. Restart if prompted.
2. **Manage Jenkins → Tools → JDK installations → Add JDK:** name `JDK21`; use the installed JDK folder, not its `bin` directory.
3. **Maven installations → Add Maven:** name `MAVEN-HOME`; use the installed Maven folder, not `bin`. Untick **Install automatically** when using local installations.
4. Check **Git installations** and click **Save**. The Jenkins service account must be able to read the installed tools.

Paths detected on this PC (use your own paths on another PC):

| Tool | Installation path |
|---|---|
| JDK | `C:UsersgayazToolsjenkinsjavajdk-21.0.13+9` |
| Maven | `C:UsersgayazToolsapache-maven-3.9.16` |
| Git executable | `C:Program FilesGitcmdgit.exe` |

With no source tests, Maven may print **No tests to run**. This is not evidence that application tests passed. A Maven build also does not compile JSPs in the same way Tomcat does at request time, so check the deployed page as well.

#### Step 1 Create the Pipeline job

**Dashboard → New Item** → name `Set3_Build` → **Pipeline → OK**. Under **Pipeline → Definition**, choose **Pipeline script**. Paste the complete Groovy script below and **Save**; storing it in the Jenkins UI avoids adding a Jenkinsfile to the original repository.

This is a **Scripted Pipeline** for a Windows Jenkins agent: it uses `node`, `tool`, `stage`, and `bat`. The checked-out project contains its POM at the root; the script places that checkout in `app` deliberately. Tool names must match **Manage Jenkins → Tools** exactly.

```groovy
node {
    def jdkHome = tool name: 'JDK21', type: 'hudson.model.JDK'
    def mvnHome = tool name: 'MAVEN-HOME', type: 'hudson.tasks.Maven$MavenInstallation'
    withEnv(["JAVA_HOME=${jdkHome}", "PATH+JDK=${jdkHome}/bin", "PATH+MAVEN=${mvnHome}/bin"]) {
        try {
            stage('Workspace cleanup') {
                deleteDir()
            }
            dir('app') {
                stage('Checkout') {
                    git branch: 'master', url: 'https://github.com/anujyog1/carrepo.git'
                }
                stage('Clean') {
                    bat 'call mvn -B clean'
                }
                stage('Build') {
                    bat 'call mvn -B compile'
                }
                stage('Test') {
                    bat 'call mvn -B test'
                }
                stage('Package') {
                    bat 'call mvn -B package'
                }
                stage('Archive') {
                    archiveArtifacts artifacts: 'target/*.war', fingerprint: true
                }
            }
        } catch (err) {
            currentBuild.result = 'FAILURE'
            throw err
        } finally {
            junit testResults: 'app/target/surefire-reports/*.xml', allowEmptyResults: true
        }
    }
}
```

#### Step 2 Build and view the stages

1. **Build Now → latest build → Console Output**; follow Checkout → Clean → Build → Test → Package → Archive.
2. Return to the job page; **Stage View** displays the named stages. Install **Pipeline: Stage View** if it is missing; the **Build Pipeline** plugin is for Freestyle chains and will not visualize this Pipeline job.
3. On a successful build, open the archived artifact `bookcar.war`.
4. Look for Maven BUILD SUCCESS and Jenkins Finished: SUCCESS.

A nonzero `bat` exit throws an error; the `catch` marks failure and rethrows it, so later normal stages are skipped. `finally` still publishes any test report. Empty reports are allowed here because the supplied repository has no Java tests; do not describe this as a passed test suite.

**Troubleshooting:** verify the exact tool names and branch, inspect Console Output for the first Maven error, and confirm the `dir(...)` block contains `pom.xml`. If an older Maven installation selects an incompatible WAR plugin, declare `org.apache.maven.plugins:maven-war-plugin` version `3.4.0` under `<build><plugins>` in a local lab POM; do not patch the original GitHub repository.

---

## Q3 Email notifications and GitHub webhooks

### Question

> Q3. a. A client requires automatic email notifications on build success or failure of Jenkins scripted pipeline, constructed for project from GitHub with link https://github.com/anujyog1/carrepo.git. Demonstrate how you will configure Jenkins email notifications on successful and failed builds. [10 Marks]
>
> Demonstrate how Jenkins pipeline triggered automatically via GitHub Webhook without manual or SCM polling. [15 Marks]
>
> Expected Output:
>
> - Webhook setup screenshot.
>
> - Illustration of auto trigger on code change in github
>
> - Jenkins console output.

**One-line explanation:** Configure SMTP notifications for both build outcomes and use a GitHub push webhook to trigger Jenkins.

### Solution

#### Step 1 Configure SMTP and Email Extension

1. Complete Q2’s working Pipeline first and install **Email Extension** from **Manage Jenkins → Plugins**.
2. Open **Manage Jenkins → System → Extended E-mail Notification**.
3. Enter the SMTP host supplied by your mail provider, for example `smtp.YOUR_PROVIDER`, and its documented port/security combination, commonly **587 with STARTTLS** or **465 with SSL**; do not tick incompatible SSL/TLS modes together.
4. Expand **Advanced**; enable SMTP authentication and add/select the sender account’s Jenkins credential. Use the provider-required app password/token rather than putting a password into the Pipeline.
5. Set **Default user e-mail suffix** if needed; set **System Admin e-mail address** under Jenkins Location to the authorized sender. Save. The built-in **E-mail Notification** section and **Extended E-mail Notification** are separate plugin settings; configure the section used by `emailext`.
6. If a test-email option is available, use your own address. Check spam and provider logs if necessary. Do not assume SMTP works until a message actually arrives.

#### Step 2 Use success and failure notification handling

Open **Set3_Build → Configure → Pipeline script** and replace the Q2 script with this complete version. Replace `YOUR_EMAIL` with your own lab recipient address:

```groovy
node {
    def jdkHome = tool name: 'JDK21', type: 'hudson.model.JDK'
    def mvnHome = tool name: 'MAVEN-HOME', type: 'hudson.tasks.Maven$MavenInstallation'
    def result = 'SUCCESS'
    withEnv(["JAVA_HOME=${jdkHome}", "PATH+JDK=${jdkHome}/bin", "PATH+MAVEN=${mvnHome}/bin"]) {
        try {
            stage('Workspace cleanup') {
                deleteDir()
            }
            dir('app') {
                stage('Checkout') {
                    git branch: 'master', url: 'https://github.com/anujyog1/carrepo.git'
                }
                stage('Clean') {
                    bat 'call mvn -B clean'
                }
                stage('Build') {
                    bat 'call mvn -B compile'
                }
                stage('Test') {
                    bat 'call mvn -B test'
                }
                stage('Package') {
                    bat 'call mvn -B package'
                }
                stage('Archive') {
                    archiveArtifacts artifacts: 'target/*.war', fingerprint: true
                }
            }
        } catch (err) {
            currentBuild.result = 'FAILURE'
            result = 'FAILURE'
            throw err
        } finally {
            junit testResults: 'app/target/surefire-reports/*.xml', allowEmptyResults: true
            emailext to: 'YOUR_EMAIL',
                subject: "${env.JOB_NAME} #${env.BUILD_NUMBER}: ${result}",
                body: "Result: ${result}\nBuild: ${env.BUILD_URL}"
        }
    }
}
```

The `catch` records failure and rethrows the build error, while `finally` sends the result for both successful and failed builds. SMTP errors appear in Console Output and must be diagnosed separately from Maven compilation errors.

#### Step 3 Demonstrate both results using local job configuration

1. Save and run **Build Now**; confirm an archived WAR and an email whose subject ends **SUCCESS**.
2. Temporarily add `error('Intentional notification test')` inside the `Build` stage, before its Maven command. Save and build; later normal stages should stop, the result should be **FAILURE**, and the failure email should arrive.
3. Remove that intentional error, save and rebuild. These changes affect only the Jenkins job script, not GitHub source.
4. If the examiner wants an actual Java compilation failure, the inspected repository has no Java sources: use a temporary Java file in the Jenkins workspace or an authorized lab copy, and label that demonstration accurately.

#### Step 4 Demonstrate the webhook part of this question

#### Configure the GitHub webhook without timed polling

These are instructions for **your own authorized lab repository**. No webhook, fork, commit or push has been created by this answer. If you cannot administer a repository, its owner must perform the GitHub-side steps; a webhook to someone else’s repository cannot be configured from Jenkins alone.

1. For a reproducible push demo, use a personal fork/lab copy of the question’s project; update the Jenkins checkout URL to that copy, keeping the correct branch.
2. In **Set3_Build → Configure → Build Triggers**, tick **GitHub hook trigger for GITScm polling**. Untick scheduled **Poll SCM** and **Build periodically** → **Save**. The trigger’s name includes “polling”, but the poll is initiated by the webhook rather than a cron schedule.
3. Download/install the Windows agent from [ngrok](https://ngrok.com/download), sign in to the ngrok dashboard, copy your agent authtoken, and run `ngrok config add-authtoken YOUR_TOKEN` in PowerShell once. Then run `ngrok http 8080`; use the HTTPS forwarding domain it prints and leave that terminal running. Keep the actual token out of screenshots and source files.
4. In **your lab GitHub repository → Settings → Webhooks → Add webhook**, set **Payload URL:** `https://YOUR_NGROK_DOMAIN/github-webhook/` including the trailing slash; **Content type:** `application/json`; choose **Just the push event**, tick **Active**, and save. If you configure a webhook secret, configure the matching verification secret in your Jenkins GitHub integration.
5. Run one manual Jenkins build to establish the checkout/repository association. In **GitHub → Webhooks → Recent Deliveries**, verify a successful response, and confirm that ngrok shows the incoming request.
6. In your lab copy only, change a harmless line such as README text on the configured branch, commit and push. Do not click **Build Now** for this second run.
7. Confirm a new Jenkins build appears with a GitHub push cause, Console Output shows the new revision, and the appropriate Pipeline/Build Pipeline view updates.
8. Capture the webhook configuration, successful delivery, Jenkins build cause, and Console Output. Do not expose tokens or credentials in screenshots.

**If it does not trigger:** check ngrok is still forwarding to local port 8080, the current HTTPS domain matches the webhook, the endpoint ends `/github-webhook/`, Jenkins is reachable, and the repository/branch matches. A restarted free ngrok tunnel may have a different address. `localhost` cannot be reached directly by GitHub. An already public Jenkins server does not need ngrok. [Jenkins GitHub plugin](https://plugins.jenkins.io/github/)

**Expected result:** successful and failed builds produce the appropriate email through the configured SMTP account; a push to an authorized lab repository starts a build automatically. Capture the webhook page/delivery, push-caused build and console output as requested. The screenshots must be taken from your own actual run, not fabricated. [Email Extension steps](https://www.jenkins.io/doc/pipeline/steps/email-ext/)

---

## Q4 Car application on AWS EC2

### Question

> Q4. As a team leader, help your teammate to deploy an application from github with link https://github.com/anujyog1/carrepo.git on an AWS EC2 Ubuntu instance. To show the deployment using public ip address, help him to launch EC2 instance, install docker and other required software’s, clone application, write the corresponding dockerfile, build the docker image and run it. [25 Marks]

**One-line explanation:** Package the car JSP application into a Tomcat image and access its context through the EC2 public IP.

### Solution

#### Step 1 Launch and connect to Ubuntu EC2

1. **AWS Console → search EC2 → Instances → Launch instances**; choose your lab region and name `set3-app`.
2. **Application and OS Images → Ubuntu Server 24.04 LTS → 64-bit x86**. Choose the small instance allowed by your lab/account, for example `t3.micro`; use `t2.micro` if your paper/lab specifically requires it and it is offered. More memory may be needed for Maven builds. Free Tier eligibility depends on your account and selected resources; do not assume a machine is always free.
3. **Key pair → Create new key pair** → name `set3-key` → RSA → `.pem`; download and keep the key.
4. **Network settings → Edit:** use a public subnet with an Internet Gateway route; enable **Auto-assign public IP**; create a Security Group with **SSH TCP 22 → My IP**; add **Custom TCP 8080** for the application. For web ports, use **My IP** for your demo or allow public HTTP access when required by the question.
5. Launch; wait for **Running** and all status checks to pass. Select the instance → **Details → Public IPv4 address**; copy it.
6. **Connect → SSH client** shows the SSH command. In Windows PowerShell, switch to the folder containing the downloaded key and run:

```powershell
Set-Location "$HOME\Downloads"
ssh -i .\set3-key.pem ubuntu@PUBLIC_IP
```

Accept the new host key when prompted. If SSH times out, recheck port 22, your current My IP, public IP and subnet route; if it says `Permission denied (publickey)`, verify the key pair and username `ubuntu`. Inbound web access is a separate rule from SSH.

If Windows SSH specifically reports overly open private-key permissions, run these commands in the key folder, then retry SSH:

```powershell
$keyPath = (Resolve-Path '.\set3-key.pem').Path
$keyAccount = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
icacls $keyPath /inheritance:r
icacls $keyPath /grant:r "${keyAccount}:(R)"
```

#### Step 2 Install required tools on EC2

**Ubuntu SSH terminal:**

```bash
sudo apt-get update
sudo apt-get install -y docker.io git curl openjdk-21-jdk
sudo systemctl enable --now docker
sudo docker --version
java -version
```

The Docker build stage provides Maven. All following Linux commands run on EC2, not in Windows PowerShell. Leave outbound internet access available for packages, Git, images and Maven dependencies. [AWS EC2 guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html)

#### Step 3 Clone the verified branch

```bash
cd ~
git clone --branch master https://github.com/anujyog1/carrepo.git
cd carrepo
git branch --show-current
ls -l pom.xml
find src -type f
```

#### Step 4 Build from source with a multi-stage Dockerfile

Run `nano Dockerfile` in this repository root; paste the following, press **Ctrl+O → Enter** to save, then **Ctrl+X** to exit:

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml ./
COPY src ./src
RUN mvn -B clean package

FROM tomcat:9.0-jdk21-temurin
COPY --from=build /app/target/bookcar.war /usr/local/tomcat/webapps/bookcar.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

Create `.dockerignore` beside it with these contents:

```text
.git
target
.settings
.classpath
.project
```

Maven produces `target/bookcar.war`; the image copies it as `bookcar.war`, making the URL context `/bookcar/`. Tomcat 9 supports these repositories’ `javax.servlet`/JSP projects; do not substitute Tomcat 10/11 without a Jakarta migration. `EXPOSE` documents a port; `-p` in the next command publishes it.

#### Step 5 Build and run

```bash
sudo docker build -t set3-app:1.0 .
sudo docker run -d --name set3-app --restart unless-stopped -p 8080:8080 set3-app:1.0
sudo docker ps --filter name=set3-app
sudo docker logs --tail 100 set3-app
curl -I http://localhost:8080/bookcar/
```

The final `.` in `docker build` is required. Wait for Tomcat to deploy the WAR; an HTTP success response from the local URL confirms application access.

#### Step 6 Verify from your laptop browser

Open **`http://PUBLIC_IP:8080/bookcar/`**, replacing `PUBLIC_IP`. The path is case-sensitive. Check the home page and the supplied JSP links (`index.jsp`, `Menu.jsp`, `register.jsp`). Only demonstrate features actually implemented; a placeholder or client-side form is not a persistent database workflow.

#### Troubleshooting and reboot behaviour

- Local `curl` works but the browser times out: check inbound TCP 8080, the current public IP, public subnet routing, and `sudo ufw status` if Ubuntu’s firewall is enabled.
- HTTP 404: confirm `/bookcar/`, inspect `sudo docker exec set3-app ls -l /usr/local/tomcat/webapps`, and read Tomcat logs.
- Container exited: run `sudo docker ps -a --filter name=set3-app` and `sudo docker logs set3-app`; do not assume Running EC2 means Running Tomcat.
- Maven errors: read the first `[ERROR]`; confirm `pom.xml`, packaging `war`, dependency connectivity and adequate memory. These JSP-only projects may report **No tests to run**.
- For a new image version, stop/remove only `set3-app`, rebuild, and rerun it. `sudo systemctl enable docker` and the restart policy restore the container after reboot unless explicitly stopped.
- A normal reboot usually retains the public IP; **stop/start** can change an auto-assigned public IP. Use an Elastic IP when a stable public IPv4 address is required.

**Expected result:** the fresh WAR runs on Tomcat and opens at `http://PUBLIC_IP:8080/bookcar/`. After your demonstration, stop/terminate the dedicated lab instance when you are finished with it.

---

## Verification and sources

**Evidence checked:** fresh repository clone, branch/commit, source tree, XML structure of the POM, packaging/final artifact name, JSP names or Java main class, and whether source tests are present. The original repository was not modified. Complete Compose configurations passed `docker compose config --quiet`; embedded YAML and Python/PowerShell syntax, navigation links, fences and practical/scenario coverage were checked. Docker tags and pinned Python package versions were confirmed through read-only registry metadata.

**Execution limits:** Jenkins jobs, email delivery, webhooks, AWS instances, Minikube workloads and third-party containers were not created or run to prepare this file. Batch Maven execution was rejected by automatic approval review because it would execute third-party build code across multiple repositories with network access; these remaining sets therefore do not claim verified Maven build success. The Expected result sections describe what to verify in your practical run.

Useful primary references:

- [Maven build lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html) and [Jenkins Scripted Pipeline syntax](https://www.jenkins.io/doc/book/pipeline/syntax/#scripted-pipeline).
- [Jenkins GitHub trigger](https://plugins.jenkins.io/github/) and [Build Pipeline for Freestyle jobs](https://plugins.jenkins.io/build-pipeline-plugin/).
- [Compose networking](https://docs.docker.com/compose/how-tos/networking/) and [startup readiness](https://docs.docker.com/compose/how-tos/startup-order/).
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/), [Minikube access](https://minikube.sigs.k8s.io/docs/handbook/accessing/) and [HPA requirements](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/).
- [Nagios Docker image paths and defaults](https://github.com/JasonRivers/Docker-Nagios), [configuration validation](https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/4/en/verifyconfig.html), and [NCPA for real remote host metrics](https://www.nagios.org/ncpa/).
- [AWS EC2 launch/connect](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html), [AWS user data behaviour](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html), and [Tomcat official image](https://hub.docker.com/_/tomcat).

