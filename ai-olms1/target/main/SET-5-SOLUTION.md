# SET 5 Software Engineering Lab Internal II Solutions

**Original paper:** `SET-5.docx` in the MORNING folder.  
**Format:** Question → One-line explanation → Solution → Expected result; scenario questions receive one-line answers.

## Quick navigation

- [Before starting](#before-starting)
- [Q1 Jenkins Scripted Pipeline and error handling](#q1-jenkins-scripted-pipeline-and-error-handling)
- [Q2 Blood Bank application on AWS EC2](#q2-blood-bank-application-on-aws-ec2)
- [Q3 Minikube and Nagios monitoring](#q3-minikube-and-nagios-monitoring)
- [Q4 Web and PostgreSQL with Compose](#q4-web-and-postgresql-with-compose)
- [Verification and sources](#verification-and-sources)

## Before starting

- **Local commands:** Windows PowerShell. **EC2 commands:** Ubuntu SSH terminal. Code blocks labelled YAML, Dockerfile, Java, Python, XML or Groovy are file contents unless explicitly described otherwise.
- Start Docker Desktop with its Linux engine. Check `docker version`, `docker compose version`, `git --version`, `java -version`, `mvn -version`, `kubectl version --client`, and `minikube version` as required by the question. Install Minikube using its [official guide](https://minikube.sigs.k8s.io/docs/start/) if missing, then reopen PowerShell.
- Keep practice files under `$HOME\se-lab-work\set5`; this answer file stays directly in the MORNING folder. Use a separate subfolder for each question.
- Jenkins is assumed to be installed and running: open [Jenkins](http://localhost:8080) and sign in with your existing account. Avoid mapping another local container to host port 8080 while Jenkins uses it; a question that explicitly requires 8080 explains how to handle the conflict.
- Replace placeholders such as `PUBLIC_IP`, `YOUR_EMAIL`, and `YOUR_NGROK_DOMAIN`. Use spaces in YAML, not tabs. Enable **File Explorer → View → Show → File name extensions** to avoid `.yml.txt` files.

### Repository checked for this paper

[Open the original GitHub repository](https://github.com/sarasrija/BloodBanksystem). Fresh read-only inspection on **20 November 2026** found:

| Item | Verified value |
|---|---|
| Branch | `main` |
| Inspected commit | `c4ac661abfa4b147d1e62c1f13b0b5df941eaa8c` |
| Maven project root | Repository root, beside `pom.xml` |
| Build output | `target/bloodbank.war` |
| Source tests | None in `src/test` |

**Important:** WAR with JSP pages; `inventory.jsp` is a work-in-progress placeholder, and no Java unit tests are included. The home page also links to `donor`, whose handler is absent. PostgreSQL in Q4 demonstrates orchestration; the supplied JSPs do not implement a PostgreSQL integration.

Repository inspection did not change GitHub. Webhook demonstrations below require a **lab repository you own or are authorized to administer**; use your own fork/copy and its URL rather than editing the question paper’s repository.

---

## Q1 Jenkins Scripted Pipeline and error handling

### Question

> 1. Jenkins – Scripted Pipeline & Build Triggers [25 MARKS]
>
> The blood bank organization is setting up a scripted Jenkins pipeline to automate compilation, unit testing, and packaging for the Blood Bank Management System repository. The team needs to handle variable folder names, manage test failures, and execute builds based on specific project requirements.
>
> Task: Write and configure a scripted Jenkins pipeline with multi-stage build tasks and error handling.
>
> Configure pipeline script with tool declarations and stage blocks.
>
> Execute Maven goals (clean, install, test, package) within batch steps.
>
> Expected Output:
>
> Console execution logs showing stage progression and artifact packaging

**One-line explanation:** Use a Scripted Jenkins Pipeline with tools, explicit Maven stages and failure handling to archive the Blood Bank WAR.

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

**Dashboard → New Item** → name `Set5_Build` → **Pipeline → OK**. Under **Pipeline → Definition**, choose **Pipeline script**. Paste the complete Groovy script below and **Save**; storing it in the Jenkins UI avoids adding a Jenkinsfile to the original repository.

This is a **Scripted Pipeline** for a Windows Jenkins agent: it uses `node`, `tool`, `stage`, and `bat`. The checked-out project contains its POM at the root; the script places that checkout in `bloodbankapp` deliberately. Tool names must match **Manage Jenkins → Tools** exactly.

```groovy
node {
    def jdkHome = tool name: 'JDK21', type: 'hudson.model.JDK'
    def mvnHome = tool name: 'MAVEN-HOME', type: 'hudson.tasks.Maven$MavenInstallation'
    withEnv(["JAVA_HOME=${jdkHome}", "PATH+JDK=${jdkHome}/bin", "PATH+MAVEN=${mvnHome}/bin"]) {
        try {
            stage('Workspace cleanup') {
                deleteDir()
            }
            dir('bloodbankapp') {
                stage('Checkout') {
                    git branch: 'main', url: 'https://github.com/sarasrija/BloodBanksystem.git'
                }
                stage('Clean') {
                    bat 'call mvn -B clean'
                }
                stage('Install') {
                    bat 'call mvn -B install'
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
            junit testResults: 'bloodbankapp/target/surefire-reports/*.xml', allowEmptyResults: true
        }
    }
}
```

#### Step 2 Build and view the stages

1. **Build Now → latest build → Console Output**; follow Checkout → Clean → Install → Test → Package → Archive.
2. Return to the job page; **Stage View** displays the named stages. Install **Pipeline: Stage View** if it is missing; the **Build Pipeline** plugin is for Freestyle chains and will not visualize this Pipeline job.
3. On a successful build, open the archived artifact `bloodbank.war`.
4. Look for Maven BUILD SUCCESS and Jenkins Finished: SUCCESS.

A nonzero `bat` exit throws an error; the `catch` marks failure and rethrows it, so later normal stages are skipped. `finally` still publishes any test report. Empty reports are allowed here because the supplied repository has no Java tests; do not describe this as a passed test suite.

`mvn install` already executes test/package and installs the artifact locally; the later Test and Package stages repeat those phases to match the question’s requested sequence. `deleteDir()` clears the whole workspace; `mvn clean` only removes Maven output such as `target/`.

**Troubleshooting:** verify the exact tool names and branch, inspect Console Output for the first Maven error, and confirm the `dir(...)` block contains `pom.xml`. If an older Maven installation selects an incompatible WAR plugin, declare `org.apache.maven.plugins:maven-war-plugin` version `3.4.0` under `<build><plugins>` in a local lab POM; do not patch the original GitHub repository.

### Scenario-based questions and one-line answers

1. **The project folder name is not mavenjava but bloodbankapp—which parts of the script must you edit?**  
   Change the Pipeline’s `dir(...)` value and any hard-coded workspace/report/artifact paths containing `mavenjava` to `bloodbankapp`; the supplied script already uses that directory.

2. **If the install stage fails, will the subsequent test and package stages still run in this scripted pipeline?**  
   No: a failed `bat` step throws an error and the normal later Test/Package stages are skipped unless error handling deliberately catches and continues.

3. **Your test cases failed, but the pipeline still continued (or failed depending on configuration)—how does Jenkins handle step execution in the test stage?**  
   A nonzero Maven test exit normally fails the `bat` step; `returnStatus`, `catchError` or ignored test failures can allow continuation, so inspect the configured error handling.

4. **If you want to print the current build number inside the console logs for auditing, how do you reference it in the pipeline script?**  
   Use `echo "Build number: ${env.BUILD_NUMBER}"` inside the Scripted Pipeline.

5. **What is the role of Jenkins in a Continuous Integration pipeline?**  
   Jenkins orchestrates source retrieval, automated builds/tests, artifact collection and result reporting so integration problems are detected early.

---

## Q2 Blood Bank application on AWS EC2

### Question

> 2. AWS – Cloud Deployment using EC2 [25 MARKS]
>
> The healthcare institution is migrating the Blood Bank Management System to an AWS EC2 instance. Administrators need to deploy static pages or Java web applications, handle local file directory paths, and automate setup routines during instance reboots.
>
> Task: Deploy and configure web assets and application archives on an AWS EC2 virtual machine.
>
> Provision Ubuntu EC2 instance, install required tools via SSH.
>
> Configure web server paths and deployment directories.
>
> Expected Output:
>
> Operational web application hosted on AWS EC2 instance.

**One-line explanation:** Deploy the Blood Bank WAR to Tomcat on EC2 and distinguish Java webapps from separately hosted static assets.

### Solution

#### Step 1 Launch and connect to Ubuntu EC2

1. **AWS Console → search EC2 → Instances → Launch instances**; choose your lab region and name `set5-app`.
2. **Application and OS Images → Ubuntu Server 24.04 LTS → 64-bit x86**. Choose the small instance allowed by your lab/account, for example `t3.micro`; use `t2.micro` if your paper/lab specifically requires it and it is offered. More memory may be needed for Maven builds. Free Tier eligibility depends on your account and selected resources; do not assume a machine is always free.
3. **Key pair → Create new key pair** → name `set5-key` → RSA → `.pem`; download and keep the key.
4. **Network settings → Edit:** use a public subnet with an Internet Gateway route; enable **Auto-assign public IP**; create a Security Group with **SSH TCP 22 → My IP**; add **Custom TCP 8080** for the application. For web ports, use **My IP** for your demo or allow public HTTP access when required by the question.
5. Launch; wait for **Running** and all status checks to pass. Select the instance → **Details → Public IPv4 address**; copy it.
6. **Connect → SSH client** shows the SSH command. In Windows PowerShell, switch to the folder containing the downloaded key and run:

```powershell
Set-Location "$HOME\Downloads"
ssh -i .\set5-key.pem ubuntu@PUBLIC_IP
```

Accept the new host key when prompted. If SSH times out, recheck port 22, your current My IP, public IP and subnet route; if it says `Permission denied (publickey)`, verify the key pair and username `ubuntu`. Inbound web access is a separate rule from SSH.

If Windows SSH specifically reports overly open private-key permissions, run these commands in the key folder, then retry SSH:

```powershell
$keyPath = (Resolve-Path '.\set5-key.pem').Path
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
git clone --branch main https://github.com/sarasrija/BloodBanksystem.git
cd BloodBanksystem
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
COPY --from=build /app/target/bloodbank.war /usr/local/tomcat/webapps/bloodbank.war
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

Maven produces `target/bloodbank.war`; the image copies it as `bloodbank.war`, making the URL context `/bloodbank/`. Tomcat 9 supports these repositories’ `javax.servlet`/JSP projects; do not substitute Tomcat 10/11 without a Jakarta migration. `EXPOSE` documents a port; `-p` in the next command publishes it.

#### Step 5 Build and run

```bash
sudo docker build -t set5-app:1.0 .
sudo docker run -d --name set5-app --restart unless-stopped -p 8080:8080 set5-app:1.0
sudo docker ps --filter name=set5-app
sudo docker logs --tail 100 set5-app
curl -I http://localhost:8080/bloodbank/
```

The final `.` in `docker build` is required. Wait for Tomcat to deploy the WAR; an HTTP success response from the local URL confirms application access.

#### Step 6 Verify from your laptop browser

Open **`http://PUBLIC_IP:8080/bloodbank/`**, replacing `PUBLIC_IP`. The path is case-sensitive. Check the home page and the supplied JSP links (`index.jsp`, `inventory.jsp`). Only demonstrate features actually implemented; a placeholder or client-side form is not a persistent database workflow.

#### Troubleshooting and reboot behaviour

- Local `curl` works but the browser times out: check inbound TCP 8080, the current public IP, public subnet routing, and `sudo ufw status` if Ubuntu’s firewall is enabled.
- HTTP 404: confirm `/bloodbank/`, inspect `sudo docker exec set5-app ls -l /usr/local/tomcat/webapps`, and read Tomcat logs.
- Container exited: run `sudo docker ps -a --filter name=set5-app` and `sudo docker logs set5-app`; do not assume Running EC2 means Running Tomcat.
- Maven errors: read the first `[ERROR]`; confirm `pom.xml`, packaging `war`, dependency connectivity and adequate memory. These JSP-only projects may report **No tests to run**.
- For a new image version, stop/remove only `set5-app`, rebuild, and rerun it. `sudo systemctl enable docker` and the restart policy restore the container after reboot unless explicitly stopped.
- A normal reboot usually retains the public IP; **stop/start** can change an auto-assigned public IP. Use an Elastic IP when a stable public IPv4 address is required.

**Expected result:** the fresh WAR runs on Tomcat and opens at `http://PUBLIC_IP:8080/bloodbank/`. After your demonstration, stop/terminate the dedicated lab instance when you are finished with it.

#### Additional static-assets option when the examiner asks for index.html

The WAR above is the actual Blood Bank repository deployment. If asked to demonstrate a separate static page under `/var/www/html`, native Ubuntu Nginx can serve it on port 80 while Tomcat uses 8080:

```bash
sudo apt-get install -y nginx
sudo systemctl enable --now nginx
echo '<h1>Blood Bank static deployment demonstration</h1>' | sudo tee /var/www/html/index.html
curl -I http://localhost/
```

Add inbound HTTP 80 to the instance Security Group and open `http://PUBLIC_IP/`. This demonstrates static assets; it is not a replacement for deploying a WAR to Tomcat’s webapps directory. For tasks on every reboot, use an enabled systemd service or configure cloud-init per-boot execution; default EC2 user data shell scripts normally run only on the first launch. [AWS user data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)

### Scenario-based questions and one-line answers

1. **You deployed your index.html or static assets to /var/www/html/ directory on EC2, but the web page still isn’t loading. What are two possible issues you would check?**  
   Check that a web server is running and serving `/var/www/html`, and that the Security Group/firewall permits its published HTTP port.

2. **You have a simple index.html file on your laptop and you launched an EC2 instance. The instance is running, but when you open the public IP in a browser, the page doesn’t load. What initial steps will you take?**  
   Install/start a web server, upload `index.html` into its document root, allow inbound HTTP 80, and verify local curl before browsing the current public IP.

3. **You can access your web application locally on the EC2 instance using curl localhost:8080, but not from your external browser. What AWS setting is likely missing or misconfigured?**  
   The Security Group must allow inbound TCP 8080 from your client; also verify public routing and any OS firewall if that rule is already correct.

4. **You want to automate the deployment of your web assets whenever you restart the EC2 instance. Which EC2 feature can be used to run setup commands automatically during instance startup?**  
   Use EC2 user data/cloud-init configured for per-boot execution or an enabled systemd service; ordinary user-data shell scripts normally run only on first launch.

5. **Why should SSH access (Port 22) be restricted in security group settings?**  
   SSH controls administration, so restrict port 22 to trusted IPs to reduce exposure to unauthorized login attempts.

---

## Q3 Minikube and Nagios monitoring

### Question

> 3. Kubernetes & Nagios – Minikube Pods & Infrastructure Monitoring [25 MARKS]
>
> During emergency blood requisition surges, system administrators use Minikube to deploy containerized scaling groups and configure Nagios to monitor server uptime, network devices, and application health checks.
>
> Task: Manage deployments in Minikube and configure Nagios host/service checks.
>
> Initialize Minikube cluster and manage pod lifecycles.
>
> Run Nagios container and review host status dashboards.
>
> Expected Output:
>
> Scaled Kubernetes deployments and active Nagios monitoring console.

**One-line explanation:** Deploy the Blood Bank image on Minikube, scale and inspect Pods, and configure Nagios host/service checks.

### Solution

#### Step 1 Create the local working folder

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set5\q3" | Out-Null
Set-Location "$HOME\se-lab-work\set5\q3"
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
git clone --branch main https://github.com/sarasrija/BloodBanksystem.git app
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
COPY --from=build /app/target/bloodbank.war /usr/local/tomcat/webapps/bloodbank.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

```powershell
minikube image build -t set5-app:1.0 .
```

This builds directly into Minikube’s image store. A regular Docker Desktop build alone may not make an image visible to Minikube. The Deployment below uses `imagePullPolicy: Never` for this local image.

#### Step 4 Create deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: set5-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: set5-app
  template:
    metadata:
      labels:
        app: set5-app
    spec:
      containers:
        - name: web
          image: set5-app:1.0
          imagePullPolicy: Never
          ports:
            - containerPort: 8080
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
  name: set5-app
spec:
  type: NodePort
  selector:
    app: set5-app
  ports:
    - port: 8095
      targetPort: 8080
```

Save this file in the current working directory, then apply it:

```powershell
kubectl apply -f .\deployment.yaml
kubectl rollout status deployment/set5-app --timeout=180s
kubectl get deployments,pods,services
minikube service set5-app --url
```

The Service port is **8095**, while the container listens on **8080**. Kubernetes allocates a NodePort, normally 30000–32767; **8095 is not the nodePort**. Open the URL printed by Minikube and append `/bloodbank/` if the URL does not already include it. On Windows with the Docker driver, keep the Minikube URL command running when it creates a tunnel.

#### Step 5 Access a fixed local port and allow Nagios to check it

In a separate PowerShell window, run this and leave it running:

```powershell
kubectl port-forward --address=0.0.0.0 service/set5-app 8095:8095
```

Open **`http://localhost:8095/bloodbank/`**. Binding to `0.0.0.0` makes the forwarding endpoint reachable from the Nagios Docker container through `host.docker.internal`; allow the relevant local firewall traffic if required. For a browser-only test without container access, omit `--address=0.0.0.0`.

#### Step 6 Scale and verify automatic recovery

```powershell
kubectl scale deployment/set5-app --replicas=3
kubectl rollout status deployment/set5-app --timeout=180s
kubectl get deployment set5-app
kubectl get pods -l app=set5-app -o wide
$podName = kubectl get pods -l app=set5-app -o jsonpath='{.items[0].metadata.name}'
kubectl delete pod $podName
kubectl get pods -l app=set5-app -w
```

Wait for **3/3** ready replicas. The ReplicaSet creates a replacement for a deleted Pod; press **Ctrl+C** after observing recovery. If a container merely crashes inside a still-existing Pod, kubelet normally restarts that container according to its restart policy; it does not always create a new Pod. Restart port forwarding if the selected Pod’s replacement interrupts the forwarding session.

#### Step 7 Inspect failures

```powershell
kubectl describe deployment set5-app
kubectl describe pods -l app=set5-app
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl logs deployment/set5-app --tail=60
kubectl get services
kubectl get endpointslices -l kubernetes.io/service-name=set5-app
```

For a restarting Pod, use `kubectl logs POD_NAME --previous` and `kubectl describe pod POD_NAME`. Pending Pods commonly lack CPU/memory capacity or have scheduling/volume constraints; an image pull problem normally shows `ImagePullBackOff`, not merely Pending forever. Check events before changing resources.

**Expected Kubernetes result:** browser access works, the Service routes to ready Pods, scaling reaches 3 replicas when resources permit, and a deleted Pod is replaced. The Nagios section below monitors the forwarded HTTP endpoint. [Minikube access](https://minikube.sigs.k8s.io/docs/handbook/accessing/)

#### Nagios Step 1 Start the monitoring server

In a separate working folder for Nagios, use Windows PowerShell:

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set5\nagios" | Out-Null
Set-Location "$HOME\se-lab-work\set5\nagios"
docker pull jasonrivers/nagios:latest
docker run -d --name set5-nagios --restart unless-stopped -p 8888:80 -e NAGIOS_TIMEZONE=Asia/Kolkata jasonrivers/nagios:latest
docker ps --filter name=set5-nagios
docker logs --tail 50 set5-nagios
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

Create **`lab.cfg`** in the same folder. The application must already be reachable on Windows host port **8095** at **`/bloodbank/`**:

```text
define command {
    command_name check_lab_http
    command_line $USER1$/check_http -H $HOSTADDRESS$ -p 8095 -u /bloodbank/ -w 2 -c 5
}
define command {
    command_name check_lab_tcp
    command_line $USER1$/check_tcp -H $HOSTADDRESS$ -p 8095
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
docker cp .\check_lab_metrics.py set5-nagios:/opt/nagios/libexec/check_lab_metrics.py
docker cp .\lab.cfg set5-nagios:/opt/nagios/etc/conf.d/lab.cfg
docker exec set5-nagios /opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
```

Only after the preflight check reports **zero errors**, restart and test:

```powershell
docker restart set5-nagios
docker exec set5-nagios sh -c 'pgrep -a nagios'
docker exec set5-nagios /opt/nagios/libexec/check_http -H host.docker.internal -p 8095 -u /bloodbank/ -w 2 -c 5
```

```powershell
docker exec set5-nagios python3 /opt/nagios/libexec/check_lab_metrics.py cpu
docker exec set5-nagios python3 /opt/nagios/libexec/check_lab_metrics.py memory
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

Stop port forwarding with Ctrl+C. Remove only this question’s resources with `kubectl delete service set5-app` and `kubectl delete deployment set5-app`. Stop monitoring with `docker stop set5-nagios` when finished; do not confuse a stopped monitoring server with an application recovery.

### Scenario-based questions and one-line answers

1. **How would you debug a failed deployment in Kubernetes using kubectl diagnostic commands?**  
   Use `kubectl describe deployment`, `kubectl describe pod`, `kubectl logs` and sorted cluster events to find image, scheduling, configuration or application failures.

2. **How do you roll back a faulty deployment in Kubernetes if an update introduces bugs?**  
   Use `kubectl rollout undo deployment/NAME`, then `kubectl rollout status deployment/NAME`; a previous recorded revision must exist.

3. **How do you debug a running Pod when application-level errors occur inside the container?**  
   Read `kubectl logs POD` and `--previous` for prior crashes, inspect `kubectl describe pod POD`, and use `kubectl exec -it POD -- sh` when a shell is available.

4. **How can you check whether the Nagios monitoring daemon/service is running properly on your system?**  
   For this Docker install use `docker exec set5-nagios pgrep -a nagios`, logs and configuration validation; for a native systemd installation use `systemctl status nagios`.

5. **What are the main advantages of using Nagios for IT infrastructure monitoring?**  
   Nagios centralizes host/service checks, configurable thresholds, alerts and recovery tracking to detect infrastructure failures promptly.

---

## Q4 Web and PostgreSQL with Compose

### Question

> 4. Docker Compose – Multi-Container Orchestration [25 MARKS
>
> The blood bank engineering team is standardizing local deployments using Docker Compose to orchestrate a web frontend container and a PostgreSQL backend container together with persistent storage volumes and custom networks.
>
> Task: Create and run a multi-container stack using a docker-compose.yml configuration file.
>
> Define services, networks, and volumes in YAML.
>
> Execute container start, stop, and clean-up commands.
>
> Expected Output:
>
> Running multi-container stack with persistent data storage and inter-service communication.

**One-line explanation:** Use Compose to run a web frontend with persistent PostgreSQL storage and service-name networking.

### Solution

#### Step 1 Create the local working folder

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set5\q4" | Out-Null
Set-Location "$HOME\se-lab-work\set5\q4"
```

Create each named file in this folder using Notepad or VS Code, paste its code, and save it. In Notepad use **Save as type → All files** and keep the exact extension. File contents below are not terminal commands.

#### Step 2 Prepare the actual web frontend

```powershell
git clone --branch main https://github.com/sarasrija/BloodBanksystem.git app
```

Create `Dockerfile` in the question folder, beside `docker-compose.yml`:

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY app/pom.xml ./
COPY app/src ./src
RUN mvn -B clean package

FROM tomcat:9.0-jdk21-temurin
COPY --from=build /app/target/bloodbank.war /usr/local/tomcat/webapps/bloodbank.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

This builds the repository’s actual JSP frontend. The shared PostgreSQL service below demonstrates a persistent backend environment; the inspected JSP source does not implement a database connection. A future JDBC implementation would use `jdbc:postgresql://db:5432/labdb`, `labuser` and the configured password; network membership alone does not add that code.

#### Step 3 Write docker-compose.yml

```yaml
name: set5-compose

services:
  web:
    build: .
    image: set5-web:1.0
    ports:
      - "8085:8080"
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

#### Step 4 Start and verify the stack

In the folder containing these files, run:

```powershell
docker compose config --quiet
docker compose up -d --build
docker compose ps
docker compose logs --tail 40 web db
docker network inspect set5-compose_app-net
docker compose exec db psql -U labuser -d labdb -c "SELECT current_database();"
```

Open **`http://localhost:8085/bloodbank/`**. Confirm the repository’s JSP home page; the SQL command should return `labdb`.

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

1. **How do you stop and clean up your running container stack and remove associated resources without losing volume data?**  
   Run `docker compose down` without `-v`; it removes the stack’s containers/network while retaining named volume data.

2. **You updated some code and want to restart your frontend, backend, and DB containers quickly. What is the easiest way to manage this lifecycle?**  
   Use `docker compose restart` for a quick restart, or `docker compose up -d --build` when changed code is baked into the application images.

3. **How do you share your application configuration and code as an image bundle with a teammate using Docker images?**  
   Export images with `docker save -o bundle.tar IMAGE...`, share that archive plus the Compose/config files, and import with `docker load -i bundle.tar`; volumes require a separate backup.

4. **What happens when you try to run two containers that expose the exact same port on the host machine?**  
   The second container cannot bind the same host IP/port, so choose different host ports or put both behind a reverse proxy.

5. **Why are networks specified in Docker Compose instead of relying solely on default bridge networking for interdependent services?**  
   Compose-managed bridge networks provide service-name DNS and project isolation; its default project network already does this, while explicit networks let you control service grouping.

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

