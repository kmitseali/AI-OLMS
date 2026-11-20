# SET 7 Software Engineering Lab Internal II Solutions

**Original paper:** `SET-7.docx` in the MORNING folder.  
**Format:** Question → One-line explanation → Solution → Expected result; scenario questions receive one-line answers.

## Quick navigation

- [Before starting](#before-starting)
- [Q1 Jenkins CI webhooks and Tomcat deployment](#q1-jenkins-ci-webhooks-and-tomcat-deployment)
- [Q2 Apartment application on AWS EC2](#q2-apartment-application-on-aws-ec2)
- [Q3 Kubernetes replicas and Nagios](#q3-kubernetes-replicas-and-nagios)
- [Q4 Tomcat and PostgreSQL with Compose](#q4-tomcat-and-postgresql-with-compose)
- [Verification and sources](#verification-and-sources)

## Before starting

- **Local commands:** Windows PowerShell. **EC2 commands:** Ubuntu SSH terminal. Code blocks labelled YAML, Dockerfile, Java, Python, XML or Groovy are file contents unless explicitly described otherwise.
- Start Docker Desktop with its Linux engine. Check `docker version`, `docker compose version`, `git --version`, `java -version`, `mvn -version`, `kubectl version --client`, and `minikube version` as required by the question. Install Minikube using its [official guide](https://minikube.sigs.k8s.io/docs/start/) if missing, then reopen PowerShell.
- Keep practice files under `$HOME\se-lab-work\set7`; this answer file stays directly in the MORNING folder. Use a separate subfolder for each question.
- Jenkins is assumed to be installed and running: open [Jenkins](http://localhost:8080) and sign in with your existing account. Avoid mapping another local container to host port 8080 while Jenkins uses it; a question that explicitly requires 8080 explains how to handle the conflict.
- Replace placeholders such as `PUBLIC_IP`, `YOUR_EMAIL`, and `YOUR_NGROK_DOMAIN`. Use spaces in YAML, not tabs. Enable **File Explorer → View → Show → File name extensions** to avoid `.yml.txt` files.

### Repository checked for this paper

[Open the original GitHub repository](https://github.com/sarasrija/ApartmentSystem). Fresh read-only inspection on **20 November 2026** found:

| Item | Verified value |
|---|---|
| Branch | `main` |
| Inspected commit | `805d21930a33b6aaec19fe15756066e12dca440b` |
| Maven project root | Repository root, beside `pom.xml` |
| Build output | `target/apartmentsystem.war` |
| Source tests | None in `src/test` |

**Important:** WAR with `index.jsp` and `complaints.jsp`; no Java unit tests are included. The home page links to `tenant`, whose handler is absent. The Q4 PostgreSQL stack is an environment demonstration: the JSP app does not implement a PostgreSQL connection.

Repository inspection did not change GitHub. Webhook demonstrations below require a **lab repository you own or are authorized to administer**; use your own fork/copy and its URL rather than editing the question paper’s repository.

---

## Q1 Jenkins CI webhooks and Tomcat deployment

### Question

> 1. Jenkins – Web Application Build and Continuous Integration [25 MARKS]
>
> A development team working on the web-based Apartment Management System repository wants to automate compilation, unit testing, and artifact generation using Jenkins. To avoid manual polling, the team needs to configure GitHub webhooks using ngrok so that every code push automatically triggers the build pipeline and displays results in the Jenkins Build History / Pipeline View.
>
> Task: Implement an automated Continuous Integration workflow using Jenkins with GitHub Webhook integration.
>
> 1. Clone Git Hub repository and Configure Tomcat web server.
>
> 3. Deploy the application on the Web Server
>
> 4. Create the Pipeline View using Jenkins
>
> Expected Output:
>
> Jenkins files as necessary
>
> Jenkins pipeline view.

**One-line explanation:** Build and deploy the Apartment WAR through a Jenkins Pipeline triggered by pushes to an authorized lab repository.

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

#### Prepare local Tomcat before building

1. Download the [Tomcat 9 ZIP](https://tomcat.apache.org/download-90.cgi), extract it, and place the installation contents in `C:\tomcat9` so `C:\tomcat9\bin` and `C:\tomcat9\webapps` exist.
2. Edit `C:\tomcat9\conf\server.xml`; in the HTTP Connector change `port="8080"` to `port="8085"` so it can run beside Jenkins. Keep its other settings.
3. Open PowerShell and run `& 'C:\tomcat9\bin\startup.bat'`. Check `http://localhost:8085`; a root 404 is possible if root content was removed, but check Tomcat logs to confirm startup.
4. The Jenkins account needs write access to `C:\tomcat9\webapps`. The pipeline copies `apartmentsystem.war` there, and Tomcat auto-deploys it.

#### Step 1 Create the Pipeline job

**Dashboard → New Item** → name `Set7_Build` → **Pipeline → OK**. Under **Pipeline → Definition**, choose **Pipeline script**. Paste the complete Groovy script below and **Save**; storing it in the Jenkins UI avoids adding a Jenkinsfile to the original repository.

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
                    git branch: 'main', url: 'https://github.com/sarasrija/ApartmentSystem.git'
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
                stage('Deploy to Tomcat') {
                    bat 'copy /Y target\\apartmentsystem.war C:\\tomcat9\\webapps\\apartmentsystem.war'
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

1. **Build Now → latest build → Console Output**; follow Checkout → Clean → Build → Test → Package → Archive → Deploy to Tomcat.
2. Return to the job page; **Stage View** displays the named stages. Install **Pipeline: Stage View** if it is missing; the **Build Pipeline** plugin is for Freestyle chains and will not visualize this Pipeline job.
3. On a successful build, open the archived artifact `apartmentsystem.war`.
4. Open `http://localhost:8085/apartmentsystem/` and verify the deployed JSP page.

A nonzero `bat` exit throws an error; the `catch` marks failure and rethrows it, so later normal stages are skipped. `finally` still publishes any test report. Empty reports are allowed here because the supplied repository has no Java tests; do not describe this as a passed test suite.

**Troubleshooting:** verify the exact tool names and branch, inspect Console Output for the first Maven error, and confirm the `dir(...)` block contains `pom.xml`. If an older Maven installation selects an incompatible WAR plugin, declare `org.apache.maven.plugins:maven-war-plugin` version `3.4.0` under `<build><plugins>` in a local lab POM; do not patch the original GitHub repository.

#### Configure the GitHub webhook without timed polling

These are instructions for **your own authorized lab repository**. No webhook, fork, commit or push has been created by this answer. If you cannot administer a repository, its owner must perform the GitHub-side steps; a webhook to someone else’s repository cannot be configured from Jenkins alone.

1. For a reproducible push demo, use a personal fork/lab copy of the question’s project; update the Jenkins checkout URL to that copy, keeping the correct branch.
2. In **Set7_Build → Configure → Build Triggers**, tick **GitHub hook trigger for GITScm polling**. Untick scheduled **Poll SCM** and **Build periodically** → **Save**. The trigger’s name includes “polling”, but the poll is initiated by the webhook rather than a cron schedule.
3. Download/install the Windows agent from [ngrok](https://ngrok.com/download), sign in to the ngrok dashboard, copy your agent authtoken, and run `ngrok config add-authtoken YOUR_TOKEN` in PowerShell once. Then run `ngrok http 8080`; use the HTTPS forwarding domain it prints and leave that terminal running. Keep the actual token out of screenshots and source files.
4. In **your lab GitHub repository → Settings → Webhooks → Add webhook**, set **Payload URL:** `https://YOUR_NGROK_DOMAIN/github-webhook/` including the trailing slash; **Content type:** `application/json`; choose **Just the push event**, tick **Active**, and save. If you configure a webhook secret, configure the matching verification secret in your Jenkins GitHub integration.
5. Run one manual Jenkins build to establish the checkout/repository association. In **GitHub → Webhooks → Recent Deliveries**, verify a successful response, and confirm that ngrok shows the incoming request.
6. In your lab copy only, change a harmless line such as README text on the configured branch, commit and push. Do not click **Build Now** for this second run.
7. Confirm a new Jenkins build appears with a GitHub push cause, Console Output shows the new revision, and the appropriate Pipeline/Build Pipeline view updates.
8. Capture the webhook configuration, successful delivery, Jenkins build cause, and Console Output. Do not expose tokens or credentials in screenshots.

**If it does not trigger:** check ngrok is still forwarding to local port 8080, the current HTTPS domain matches the webhook, the endpoint ends `/github-webhook/`, Jenkins is reachable, and the repository/branch matches. A restarted free ngrok tunnel may have a different address. `localhost` cannot be reached directly by GitHub. An already public Jenkins server does not need ngrok. [Jenkins GitHub plugin](https://plugins.jenkins.io/github/)

### Scenario-based questions and one-line answers

1. **After setting up the GitHub webhook , you push code changes, but Jenkins does not trigger automatically. What simple check would you perform on the ngrok terminal session to ensure it is still running and forwarding requests?**  
   Confirm ngrok still displays an active HTTPS forwarding URL to `http://localhost:8080` and receives webhook requests; update your lab webhook if its domain changed.

2. **How does Jenkins display compilation errors in the console output when a syntax mistake is made in a Java file?**  
   Maven prints `[ERROR]` compiler messages with the source file and line, exits nonzero, and Jenkins marks the build FAILED in Console Output.

3. **If Maven successfully compiles files , but a unit test fails during , will Jenkins still go ahead and create the .war file?**  
   Normally no: Maven stops at the failing test phase before packaging; ignoring test failures or retaining an older WAR can obscure this, so clean and inspect the result.

4. **Why is ngrok necessary when connecting a local Jenkins server  to GitHub webhooks?**  
   ngrok supplies a public HTTPS route to local Jenkins because GitHub cannot send webhooks to your laptop’s private localhost.

5. **What is the difference between Continuous Integration and Continuous Deployment?**  
   Continuous Integration automatically builds/tests changes; Continuous Deployment also automatically releases validated changes to the target environment.

---

## Q2 Apartment application on AWS EC2

### Question

> 2. AWS – Cloud Deployment using EC2 [25 MARKS]
>
> The apartment association wants to migrate the Apartment Management System from a local server to an AWS EC2 instance so residents and administrators can access the portal remotely over the internet. The deployment requires provisioning an Ubuntu virtual machine, installing Docker and Git, writing a Dockerfile to package the application with Tomcat, and securing network access.
>
> Task: Deploy the Apartment Management application on AWS using an EC2 instance and Docker.
>
> Expected Output:
>
> Launch an Ubuntu EC2 instance (t2.micro, free tier) and connect via SSH using the .pem key.
>
> Update system packages, install Docker and Git .
>
> Clone the GitHub repository onto the instance and Write a custom Docker file.
>
> Build the Docker image, run it as a detached container mapping host port to container port 8090, and verify access via the EC2 public IP.

**One-line explanation:** Build a Tomcat image listening on container port 8090 and publish it on EC2 host port 8090.

### Solution

#### Step 1 Launch and connect to Ubuntu EC2

1. **AWS Console → search EC2 → Instances → Launch instances**; choose your lab region and name `set7-app`.
2. **Application and OS Images → Ubuntu Server 24.04 LTS → 64-bit x86**. Choose the small instance allowed by your lab/account, for example `t3.micro`; use `t2.micro` if your paper/lab specifically requires it and it is offered. More memory may be needed for Maven builds. Free Tier eligibility depends on your account and selected resources; do not assume a machine is always free.
3. **Key pair → Create new key pair** → name `set7-key` → RSA → `.pem`; download and keep the key.
4. **Network settings → Edit:** use a public subnet with an Internet Gateway route; enable **Auto-assign public IP**; create a Security Group with **SSH TCP 22 → My IP**; add **Custom TCP 8090** for the application. For web ports, use **My IP** for your demo or allow public HTTP access when required by the question.
5. Launch; wait for **Running** and all status checks to pass. Select the instance → **Details → Public IPv4 address**; copy it.
6. **Connect → SSH client** shows the SSH command. In Windows PowerShell, switch to the folder containing the downloaded key and run:

```powershell
Set-Location "$HOME\Downloads"
ssh -i .\set7-key.pem ubuntu@PUBLIC_IP
```

Accept the new host key when prompted. If SSH times out, recheck port 22, your current My IP, public IP and subnet route; if it says `Permission denied (publickey)`, verify the key pair and username `ubuntu`. Inbound web access is a separate rule from SSH.

If Windows SSH specifically reports overly open private-key permissions, run these commands in the key folder, then retry SSH:

```powershell
$keyPath = (Resolve-Path '.\set7-key.pem').Path
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
git clone --branch main https://github.com/sarasrija/ApartmentSystem.git
cd ApartmentSystem
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
RUN sed -i 's/port="8080"/port="8090"/' /usr/local/tomcat/conf/server.xml
COPY --from=build /app/target/apartmentsystem.war /usr/local/tomcat/webapps/apartmentsystem.war
EXPOSE 8090
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

Maven produces `target/apartmentsystem.war`; the image copies it as `apartmentsystem.war`, making the URL context `/apartmentsystem/`. Tomcat 9 supports these repositories’ `javax.servlet`/JSP projects; do not substitute Tomcat 10/11 without a Jakarta migration. `EXPOSE` documents a port; `-p` in the next command publishes it.

#### Step 5 Build and run

```bash
sudo docker build -t set7-app:1.0 .
sudo docker run -d --name set7-app --restart unless-stopped -p 8090:8090 set7-app:1.0
sudo docker ps --filter name=set7-app
sudo docker logs --tail 100 set7-app
curl -I http://localhost:8090/apartmentsystem/
```

The final `.` in `docker build` is required. Wait for Tomcat to deploy the WAR; an HTTP success response from the local URL confirms application access.

#### Step 6 Verify from your laptop browser

Open **`http://PUBLIC_IP:8090/apartmentsystem/`**, replacing `PUBLIC_IP`. The path is case-sensitive. Check the home page and the supplied JSP links (`complaints.jsp`, `index.jsp`). Only demonstrate features actually implemented; a placeholder or client-side form is not a persistent database workflow.

#### Troubleshooting and reboot behaviour

- Local `curl` works but the browser times out: check inbound TCP 8090, the current public IP, public subnet routing, and `sudo ufw status` if Ubuntu’s firewall is enabled.
- HTTP 404: confirm `/apartmentsystem/`, inspect `sudo docker exec set7-app ls -l /usr/local/tomcat/webapps`, and read Tomcat logs.
- Container exited: run `sudo docker ps -a --filter name=set7-app` and `sudo docker logs set7-app`; do not assume Running EC2 means Running Tomcat.
- Maven errors: read the first `[ERROR]`; confirm `pom.xml`, packaging `war`, dependency connectivity and adequate memory. These JSP-only projects may report **No tests to run**.
- For a new image version, stop/remove only `set7-app`, rebuild, and rerun it. `sudo systemctl enable docker` and the restart policy restore the container after reboot unless explicitly stopped.
- A normal reboot usually retains the public IP; **stop/start** can change an auto-assigned public IP. Use an Elastic IP when a stable public IPv4 address is required.

**Expected result:** the fresh WAR runs on Tomcat and opens at `http://PUBLIC_IP:8090/apartmentsystem/`. After your demonstration, stop/terminate the dedicated lab instance when you are finished with it.

### Scenario-based questions and one-line answers

1. **Your container is running on port 8090 inside EC2, but you try to open port 8080 in your browser URL. What type of connection error do you get, and which AWS setting must you update in the Security Group?**  
   Port 8080 has no matching listener so it is refused or times out; browse the published 8090 URL and allow inbound TCP 8090 in the Security Group.

2. **Why is leaving SSH access (Port 22) open to everyone (0.0.0.0/0) considered a security               risk in AWS Security Groups?**  
   It exposes the administrative login endpoint to the whole internet, increasing the chance of unauthorized attempts; use trusted source IPs and key authentication.

3. **What are the basic Linux shell commands used to refresh packages and install Docker on a new Ubuntu EC2 instance?**  
   Run `sudo apt-get update` followed by `sudo apt-get install -y docker.io git` and `sudo systemctl enable --now docker`.

4. **If the Tomcat server inside your running container crashes unexpectedly, how can you view the container logs to find out what went wrong?**  
   Run `sudo docker logs --tail 100 set7-app` or `sudo docker logs -f set7-app` to inspect Tomcat startup and failure messages.

5. **How can application availability be improved?**  
   Use health checks, process/container restart policies, multiple instances across Availability Zones, a load balancer, Auto Scaling and monitoring.

---

## Q3 Kubernetes replicas and Nagios

### Question

> 3. Kubernetes – Container Deployment and Auto Scaling [25 MARKS]
>
> Scenario: Managing Heavy Traffic During Monthly Maintenance Deadlines
>
> On the final day of every month, thousands of residents simultaneously log in to the Apartment Management Portal to pay monthly maintenance bills and book facilities. To handle this surge and ensure high availability, the administration wants to deploy multiple application replicas using Kubernetes (Minikube) and ensure automatic recovery when containers fail.
>
> Task: Deploy, scale, and manage the application using Kubernetes.
>
> Expected Output:
>
> Start a local Kubernetes cluster using Minikube .
>
> Create a Deployment for the apartment application with multiple 4 replicas
>
> Expose the application using a Kubernetes Service.
>
> Scale the deployment to handle higher loads.
>
> Pull and run the Nagios Docker image and run the container.
>
> Monitor CPU, memory, disk and server uptime.

**One-line explanation:** Deploy four Apartment application replicas, scale them, demonstrate recovery and configure Nagios monitoring.

### Solution

#### Step 1 Create the local working folder

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set7\q3" | Out-Null
Set-Location "$HOME\se-lab-work\set7\q3"
```

Create each named file in this folder using Notepad or VS Code, paste its code, and save it. In Notepad use **Save as type → All files** and keep the exact extension. File contents below are not terminal commands.

#### Step 2 Start the Minikube cluster

```powershell
minikube start --driver=docker
minikube status
kubectl config use-context minikube
kubectl get nodes
```

Wait for the Minikube node to be **Ready**. These commands operate on the `minikube` context; do not deploy to another cluster by accident.

#### Step 3 Build the repository image inside Minikube

```powershell
git clone --branch main https://github.com/sarasrija/ApartmentSystem.git app
Set-Location .\app
notepad Dockerfile
```

Save the following Dockerfile in that clone:

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml ./
COPY src ./src
RUN mvn -B clean package

FROM tomcat:9.0-jdk21-temurin
RUN sed -i 's/port="8080"/port="8090"/' /usr/local/tomcat/conf/server.xml
COPY --from=build /app/target/apartmentsystem.war /usr/local/tomcat/webapps/apartmentsystem.war
EXPOSE 8090
CMD ["catalina.sh", "run"]
```

```powershell
minikube image build -t set7-app:1.0 .
```

This builds directly into Minikube’s image store. A regular Docker Desktop build alone may not make an image visible to Minikube. The Deployment below uses `imagePullPolicy: Never` for this local image.

#### Step 4 Create deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: set7-app
spec:
  replicas: 4
  selector:
    matchLabels:
      app: set7-app
  template:
    metadata:
      labels:
        app: set7-app
    spec:
      containers:
        - name: web
          image: set7-app:1.0
          imagePullPolicy: Never
          ports:
            - containerPort: 8090
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: set7-app
spec:
  type: NodePort
  selector:
    app: set7-app
  ports:
    - port: 8097
      targetPort: 8090
```

Save this file in the current working directory, then apply it:

```powershell
kubectl apply -f .\deployment.yaml
kubectl rollout status deployment/set7-app --timeout=180s
kubectl get deployments,pods,services
minikube service set7-app --url
```

The Service port is **8097**, while the container listens on **8090**. Kubernetes allocates a NodePort, normally 30000–32767; **8097 is not the nodePort**. Open the URL printed by Minikube and append `/apartmentsystem/` if the URL does not already include it. On Windows with the Docker driver, keep the Minikube URL command running when it creates a tunnel.

#### Step 5 Access a fixed local port and allow Nagios to check it

In a separate PowerShell window, run this and leave it running:

```powershell
kubectl port-forward --address=0.0.0.0 service/set7-app 8097:8097
```

Open **`http://localhost:8097/apartmentsystem/`**. Binding to `0.0.0.0` makes the forwarding endpoint reachable from the Nagios Docker container through `host.docker.internal`; allow the relevant local firewall traffic if required. For a browser-only test without container access, omit `--address=0.0.0.0`.

#### Step 6 Scale and verify automatic recovery

```powershell
kubectl scale deployment/set7-app --replicas=5
kubectl rollout status deployment/set7-app --timeout=180s
kubectl get deployment set7-app
kubectl get pods -l app=set7-app -o wide
$podName = kubectl get pods -l app=set7-app -o jsonpath='{.items[0].metadata.name}'
kubectl delete pod $podName
kubectl get pods -l app=set7-app -w
```

Wait for **5/5** ready replicas. The ReplicaSet creates a replacement for a deleted Pod; press **Ctrl+C** after observing recovery. If a container merely crashes inside a still-existing Pod, kubelet normally restarts that container according to its restart policy; it does not always create a new Pod. Restart port forwarding if the selected Pod’s replacement interrupts the forwarding session.

#### Step 7 Inspect failures

```powershell
kubectl describe deployment set7-app
kubectl describe pods -l app=set7-app
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl logs deployment/set7-app --tail=60
kubectl get services
kubectl get endpointslices -l kubernetes.io/service-name=set7-app
```

For a restarting Pod, use `kubectl logs POD_NAME --previous` and `kubectl describe pod POD_NAME`. Pending Pods commonly lack CPU/memory capacity or have scheduling/volume constraints; an image pull problem normally shows `ImagePullBackOff`, not merely Pending forever. Check events before changing resources.

#### Optional extension Demonstrate real CPU autoscaling

The scale command above is **manual scaling**. To demonstrate an HPA, enable metrics and use the CPU requests already present in the YAML:

```powershell
minikube addons enable metrics-server
kubectl top pods
kubectl autoscale deployment set7-app --cpu-percent=60 --min=4 --max=5
kubectl get hpa -w
```

Wait for valid metrics; no load means the HPA may stay at its minimum. It will not necessarily reach 5 replicas without sustained CPU use. Press Ctrl+C after observing it; remove this optional HPA with `kubectl delete hpa set7-app` before expecting later manual replica counts to remain fixed. [Kubernetes HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)

**Expected Kubernetes result:** browser access works, the Service routes to ready Pods, scaling reaches 5 replicas when resources permit, and a deleted Pod is replaced. The Nagios section below monitors the forwarded HTTP endpoint. [Minikube access](https://minikube.sigs.k8s.io/docs/handbook/accessing/)

#### Nagios Step 1 Start the monitoring server

In a separate working folder for Nagios, use Windows PowerShell:

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set7\nagios" | Out-Null
Set-Location "$HOME\se-lab-work\set7\nagios"
docker pull jasonrivers/nagios:latest
docker run -d --name set7-nagios --restart unless-stopped -p 8888:80 -e NAGIOS_TIMEZONE=Asia/Kolkata jasonrivers/nagios:latest
docker ps --filter name=set7-nagios
docker logs --tail 50 set7-nagios
```

Open **`http://localhost:8888`** and log in with the image’s documented default **nagiosadmin / nagios**. Click **Hosts**, then **Services**; use the status detail pages for last check time, plugin output and state. This is Nagios Core, which checks periodically; the page is not a continuous streaming dashboard. If another set’s Nagios container already occupies 8888, stop that named container before this exercise.

#### Nagios Step 2 Add explicit CPU memory disk and uptime checks

Create **`check_lab_metrics.py`** in the Nagios working folder:

```python
#!/usr/bin/env python3
import sys, time, shutil

mode = sys.argv[1]
warning, critical = 80.0, 90.0
if mode == "cpu":
    def snapshot():
        with open("/proc/stat") as f:
            values = list(map(int, f.readline().split()[1:9]))
        return sum(values), values[3] + values[4]
    t1, idle1 = snapshot()
    time.sleep(1)
    t2, idle2 = snapshot()
    value = 100 * (1 - (idle2-idle1) / max(t2-t1, 1))
elif mode == "memory":
    with open("/proc/meminfo") as f:
        mem = {line.split(":")[0]: int(line.split()[1]) for line in f}
    value = 100 * (mem["MemTotal"]-mem["MemAvailable"]) / mem["MemTotal"]
elif mode == "disk":
    disk = shutil.disk_usage("/")
    value = 100 * disk.used / disk.total
elif mode == "uptime":
    with open("/proc/uptime") as f:
        seconds = float(f.read().split()[0])
    print(f"OK - Linux system uptime {seconds/3600:.1f} hours | uptime={seconds:.0f}s")
    sys.exit(0)
else:
    print("UNKNOWN - select cpu, memory, disk or uptime")
    sys.exit(3)
state = 2 if value >= critical else 1 if value >= warning else 0
label = ["OK", "WARNING", "CRITICAL"][state]
print(f"{label} - {mode} used {value:.1f}% | {mode}={value:.1f}%;80;90;0;100")
sys.exit(state)
```

This plugin samples actual CPU busy percentage over one second, memory used from `MemAvailable`, disk usage, and Linux uptime. The warning/critical thresholds are 80%/90%. CPU load average is a different measurement and should not be reported as CPU utilization percentage.

#### Nagios Step 3 Define the monitored endpoint

Create **`lab.cfg`** in the same folder. The application must already be reachable on Windows host port **8097** at **`/apartmentsystem/`**:

```text
define command {
    command_name check_lab_http
    command_line $USER1$/check_http -H $HOSTADDRESS$ -p 8097 -u /apartmentsystem/ -w 2 -c 5
}
define command {
    command_name check_lab_tcp
    command_line $USER1$/check_tcp -H $HOSTADDRESS$ -p 8097
}
define host {
    use linux-server
    host_name lab-application
    alias Application HTTP endpoint
    address host.docker.internal
    check_command check_lab_tcp
    check_interval 1
    retry_interval 1
    max_check_attempts 2
}
define service {
    use generic-service
    host_name lab-application
    service_description Application HTTP
    check_command check_lab_http
    check_interval 1
    retry_interval 1
    max_check_attempts 2
}
define command {
    command_name check_lab_cpu
    command_line /usr/bin/python3 /opt/nagios/libexec/check_lab_metrics.py cpu
}
define service {
    use generic-service
    host_name localhost
    service_description Linux cpu
    check_command check_lab_cpu
    check_interval 1
    retry_interval 1
    max_check_attempts 2
}
define command {
    command_name check_lab_memory
    command_line /usr/bin/python3 /opt/nagios/libexec/check_lab_metrics.py memory
}
define service {
    use generic-service
    host_name localhost
    service_description Linux memory
    check_command check_lab_memory
    check_interval 1
    retry_interval 1
    max_check_attempts 2
}
define command {
    command_name check_lab_disk
    command_line /usr/bin/python3 /opt/nagios/libexec/check_lab_metrics.py disk
}
define service {
    use generic-service
    host_name localhost
    service_description Linux disk
    check_command check_lab_disk
    check_interval 1
    retry_interval 1
    max_check_attempts 2
}
define command {
    command_name check_lab_uptime
    command_line /usr/bin/python3 /opt/nagios/libexec/check_lab_metrics.py uptime
}
define service {
    use generic-service
    host_name localhost
    service_description Linux uptime
    check_command check_lab_uptime
    check_interval 1
    retry_interval 1
    max_check_attempts 2
}
```

`host.docker.internal` reaches the Windows host from a Docker Desktop Linux container; `localhost` inside Nagios would reach Nagios itself. The host check here tests the application’s TCP endpoint; the service check tests HTTP and response time separately. Use a real server IP/DNS instead when monitoring another machine.

#### Nagios Step 4 Copy validate and load configuration

```powershell
docker cp .\check_lab_metrics.py set7-nagios:/opt/nagios/libexec/check_lab_metrics.py
docker cp .\lab.cfg set7-nagios:/opt/nagios/etc/conf.d/lab.cfg
docker exec set7-nagios /opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
```

Only after the preflight check reports **zero errors**, restart and test:

```powershell
docker restart set7-nagios
docker exec set7-nagios sh -c 'pgrep -a nagios'
docker exec set7-nagios /opt/nagios/libexec/check_http -H host.docker.internal -p 8097 -u /apartmentsystem/ -w 2 -c 5
```

```powershell
docker exec set7-nagios python3 /opt/nagios/libexec/check_lab_metrics.py cpu
docker exec set7-nagios python3 /opt/nagios/libexec/check_lab_metrics.py memory
```

The inspected image uses **`/opt/nagios`**, including `etc/conf.d`, `bin/nagios`, `libexec` and `var/status.dat`; do not mix these paths with another installation’s `/usr/local/nagios` layout. The image’s `nagios.cfg` already includes `cfg_dir=/opt/nagios/etc/conf.d`. [Image documentation](https://github.com/JasonRivers/Docker-Nagios), [Nagios configuration validation](https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/4/en/verifyconfig.html)

#### Nagios Step 5 Observe failure and recovery

1. In **Services**, find **Application HTTP**, and, when configured, **Linux cpu**, **Linux memory**, **Linux disk**, **Linux uptime** under `localhost`.
2. Wait for the next check or use **Service detail → Re-schedule the next check → Force check** if available. Read the plugin output and last-check time; a stale green page is not a new check.
3. Temporarily stop the application container, or press **Ctrl+C** in the application’s Kubernetes port-forward terminal. Confirm the endpoint changes to a failure state after its configured check/retry interval.
4. Restart the application or rerun port forwarding; force/wait for checks and confirm recovery to **OK**. Stopping the forwarding process demonstrates loss of the monitored network endpoint; it does not mean the Kubernetes Pods were deleted.
5. The dashboard state transition is the lab alert. Actual email delivery additionally requires contact definitions, notification options and a working mail relay; it is not enabled merely by viewing a red status.

**What these metrics mean:** `/proc` and `/` inside a Docker Desktop container describe the Linux/Docker environment visible to that container, not native Windows CPU, Windows RAM, the Windows filesystem, or every remote Pod. To monitor the actual Windows host or a remote Linux server, install/configure a supported agent such as NCPA on that target and configure agent checks with its address/token; do not label this local demonstration as remote host metrics. HTTP checks above really test the published application endpoint. [Nagios NCPA agent](https://www.nagios.org/ncpa/)

**If the dashboard cannot connect to the Nagios process:** inspect container logs, run the preflight command, correct errors, then restart; confirm `pgrep -a nagios` and the status file. A new host will appear only after its configuration is included, validates and is loaded. If HTTP fails only from Nagios, verify the host mapping, listening address and Windows firewall access from Docker.

**Expected result:** an active Nagios daemon, visible hosts/services, explicit metric outputs and thresholds, and an HTTP failure/recovery visible in Services.

#### Clean up after the demonstration

Stop port forwarding with Ctrl+C. Remove only this question’s resources with `kubectl delete service set7-app` and `kubectl delete deployment set7-app`. Stop monitoring with `docker stop set7-nagios` when finished; do not confuse a stopped monitoring server with an application recovery.

### Scenario-based questions and one-line answers

1. **If one of your application pods stops working suddenly due to a crash, how does Kubernetes handle it to maintain the required number of replicas?**  
   Kubelet restarts a crashed container; when a Pod is lost/deleted, the Deployment’s ReplicaSet creates a replacement to restore the desired replicas.

2. **What is the basic difference between a Pod IP address and a Service IP address when routing traffic?**  
   A Pod IP identifies one ephemeral Pod, while a Service provides stable addressing and routes to matching ready backends.

3. **Which kubectl command is used to check the list of currently running pods and their statuses?**  
   Use `kubectl get pods` or `kubectl get pods -o wide`.

4. **If you cannot open the NodePort page directly in your browser, what alternative command (kubectl port-forward) can you use to test the app locally?**  
   Use `kubectl port-forward service/set7-app 8097:8097`, then open `http://localhost:8097/apartmentsystem/`.

5. **How can Nagios identify application failure when the server is running?**  
   Use an application-level HTTP check for the correct port/path so a reachable server can still report a failed application service.

---

## Q4 Tomcat and PostgreSQL with Compose

### Question

> 4. Docker Compose – Multi-Container Application [25 MARKS]
>
> Scenario: Multi-Container Orchestration for Apartment Community Portal
>
> The development team needs to run the Apartment Management System web frontend alongside a backend PostgreSQL database (postgres:15) as an integrated stack. To streamline local development, ensure consistent environments, and handle data persistence across restarts, the team must define and execute the setup using Docker Compose.
>
> Task: Define and run interdependent multi-container services using docker-compose.yml.
>
> Expected Output:
>
> Create a working directory (compose-lab) containing a docker-compose.yml file (version 3.9).
>
> Define a web service (Nginx/Tomcat) and a db service (postgres:15) with appropriate environment variables.
>
> Configure custom container networking (networks) so services can communicate securely.
>
> Configure persistent storage volumes (volumes) so database records persist after container removal .
>
> Run the stack in detached mode , verify container status, and test rebuilding after code changes.

**One-line explanation:** Run the requested Tomcat/PostgreSQL development environment with custom networking and persistent data.

### Solution

#### Step 1 Create the local working folder

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set7\q4\compose-lab" | Out-Null
Set-Location "$HOME\se-lab-work\set7\q4\compose-lab"
```

Create each named file in this folder using Notepad or VS Code, paste its code, and save it. In Notepad use **Save as type → All files** and keep the exact extension. File contents below are not terminal commands.

#### Step 2 Prepare the actual web frontend

```powershell
git clone --branch main https://github.com/sarasrija/ApartmentSystem.git app
```

Create `Dockerfile` in the question folder, beside `docker-compose.yml`:

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY app/pom.xml ./
COPY app/src ./src
RUN mvn -B clean package

FROM tomcat:9.0-jdk21-temurin
COPY --from=build /app/target/apartmentsystem.war /usr/local/tomcat/webapps/apartmentsystem.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

This builds the repository’s actual JSP frontend. The shared PostgreSQL service below demonstrates a persistent backend environment; the inspected JSP source does not implement a database connection. A future JDBC implementation would use `jdbc:postgresql://db:5432/labdb`, `labuser` and the configured password; network membership alone does not add that code.

#### Step 3 Write docker-compose.yml

```yaml
version: "3.9"
name: set7-compose

services:
  web:
    build: .
    image: set7-web:1.0
    ports:
      - "8087:8080"
    depends_on:
      db:
        condition: service_healthy
    networks: [app-net]

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: labdb
      POSTGRES_USER: labuser
      POSTGRES_PASSWORD: LabPgPass
    volumes:
      - pg-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U labuser -d labdb"]
      interval: 5s
      timeout: 3s
      retries: 20
    networks: [app-net]

volumes:
  pg-data:

networks:
  app-net:
    driver: bridge
```

`version: "3.9"` is retained because this paper explicitly requests it; modern Compose v2 ignores it and may display an “obsolete” warning. The service configuration remains valid.

#### Step 4 Start and verify the stack

In the folder containing these files, run:

```powershell
docker compose config --quiet
docker compose up -d --build
docker compose ps
docker compose logs --tail 40 web db
docker network inspect set7-compose_app-net
docker compose exec db psql -U labuser -d labdb -c "SELECT current_database();"
```

Open **`http://localhost:8087/apartmentsystem/`**. Confirm the repository’s JSP home page; the SQL command should return `labdb`.

#### Step 5 Verify persistent database records

```powershell
docker compose exec db psql -U labuser -d labdb -c "CREATE TABLE IF NOT EXISTS persistence_demo (id integer PRIMARY KEY, note text); INSERT INTO persistence_demo VALUES (1, 'saved') ON CONFLICT (id) DO NOTHING;"
docker compose down
docker compose up -d
docker compose ps
```

Wait for the database to become healthy, then run:

```powershell
docker compose exec db psql -U labuser -d labdb -c "SELECT * FROM persistence_demo;"
```

The row should remain. `docker compose down` keeps named volumes; `down -v` removes them and therefore removes this persistence guarantee. Database environment variables initialize an empty volume; changing them later does not update existing database users.

#### Step 6 Rebuild and manage the lifecycle

```powershell
docker compose up -d --build web
docker compose restart
docker compose stop
docker compose start
```

Edit `app/src/main/webapp/index.jsp` in your local clone before the rebuild to observe a changed page. `restart` alone does not rebuild an image. To remove containers and the network while keeping records, run `docker compose down`.

**Troubleshooting:** check `docker compose ps`, service logs and `docker compose config`; inside the network use `db:5432` and `redis:6379`, never `localhost`. A shared network enables connectivity but does not itself enforce database authentication. If a host port is already occupied, choose a free host port and update the browser URL.

**Expected result:** all defined services run, the web page is reachable, SQL succeeds, and records survive container recreation.

### Scenario-based questions and one-line answers

1. **What is the main use of the depends_on setting in a docker-compose.yml file?**  
   `depends_on` controls startup order; combine `condition: service_healthy` with a database healthcheck when the frontend must wait for database readiness.

2. **Why do we configure persistent volumes for databases in Docker Compose, and what happens to saved records if volumes are left out when running docker compose down?**  
   Named volumes preserve database files when containers are removed; without persistent storage, removal of the database container loses data kept only in its writable layer.

3. **When you update your source code and run docker compose up -d, why don't the changes show up right away, and what flag should you add to rebuild everything?**  
   Changed files baked into an image need rebuilding; use `docker compose up -d --build`.

4. **How do custom networks help containers find and talk to each other using names instead of IP addresses?**  
   Containers on the same user-defined Compose network resolve service names such as `db` through Docker DNS instead of relying on changing IPs.

5. **How can database information be recovered after container deletion?**  
   Recreate MySQL/PostgreSQL with the original retained data volume mounted at its database data directory, or restore a backup if the volume was deleted.

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

