# SET 2 Software Engineering Lab Internal II Solutions

**Original paper:** `Set-2.docx` in the MORNING folder.  
**Format:** Question → One-line explanation → Solution → Expected result; scenario questions receive one-line answers.

## Quick navigation

- [Before starting](#before-starting)
- [Q1 Nginx and PostgreSQL with Compose](#q1-nginx-and-postgresql-with-compose)
- [Q2 Maven Jenkins Pipeline View and Poll SCM](#q2-maven-jenkins-pipeline-view-and-poll-scm)
- [Q3 Nagios system and application monitoring](#q3-nagios-system-and-application-monitoring)
- [Q4 Metro application on AWS EC2](#q4-metro-application-on-aws-ec2)
- [Verification and sources](#verification-and-sources)

## Before starting

- **Local commands:** Windows PowerShell. **EC2 commands:** Ubuntu SSH terminal. Code blocks labelled YAML, Dockerfile, Java, Python, XML or Groovy are file contents unless explicitly described otherwise.
- Start Docker Desktop with its Linux engine. Check `docker version`, `docker compose version`, `git --version`, `java -version`, `mvn -version`, `kubectl version --client`, and `minikube version` as required by the question. Install Minikube using its [official guide](https://minikube.sigs.k8s.io/docs/start/) if missing, then reopen PowerShell.
- Keep practice files under `$HOME\se-lab-work\set2`; this answer file stays directly in the MORNING folder. Use a separate subfolder for each question.
- Jenkins is assumed to be installed and running: open [Jenkins](http://localhost:8080) and sign in with your existing account. Avoid mapping another local container to host port 8080 while Jenkins uses it; a question that explicitly requires 8080 explains how to handle the conflict.
- Replace placeholders such as `PUBLIC_IP`, `YOUR_EMAIL`, and `YOUR_NGROK_DOMAIN`. Use spaces in YAML, not tabs. Enable **File Explorer → View → Show → File name extensions** to avoid `.yml.txt` files.

### Repository checked for this paper

[Open the original GitHub repository](https://github.com/anujyog1/MetroRepo). Fresh read-only inspection on **20 November 2026** found:

| Item | Verified value |
|---|---|
| Branch | `master` |
| Inspected commit | `242d06fa82e5c63122dd32dc5bca10a77eea6a6b` |
| Maven project root | Repository root, beside `pom.xml` |
| Build output | `target/Book-Metro-Ticket.war` |
| Source tests | None in `src/test` |

**Important:** JSP-only WAR; no Java unit tests are included. The context path is case-sensitive: `/Book-Metro-Ticket/`.

Repository inspection did not change GitHub. Webhook demonstrations below require a **lab repository you own or are authorized to administer**; use your own fork/copy and its URL rather than editing the question paper’s repository.

---

## Q1 Nginx and PostgreSQL with Compose

### Question

> Q1. A software development company wants to deploy a simple application environment consisting of an Nginx web server and a PostgreSQL database. The administrator wants to manage both services using a single Docker Compose configuration file instead of creating and running containers individually. As a developer demonstrate how you will deploy it. [20 Marks]

**One-line explanation:** Run an Nginx frontend and persistent PostgreSQL service through one Compose configuration.

### Solution

#### Step 1 Create the local working folder

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set2\q1" | Out-Null
Set-Location "$HOME\se-lab-work\set2\q1"
```

Create each named file in this folder using Notepad or VS Code, paste its code, and save it. In Notepad use **Save as type → All files** and keep the exact extension. File contents below are not terminal commands.

#### Step 2 Create a small web image

**File: `index.html`**

```html
<!doctype html>
<html><head><title>Set 2 Compose</title></head>
<body><h1>Set 2: Nginx and PostgreSQL are managed by Compose</h1></body></html>
```

**File: `Dockerfile`**

```dockerfile
FROM nginx:stable
COPY index.html /usr/share/nginx/html/index.html
```

Nginx serves this HTML page; Nginx itself does not query PostgreSQL. Both services share a network, and the database is tested with SQL below. This is an orchestration environment, not a claim that the paper’s JSP application gained a database integration.

#### Step 3 Write docker-compose.yml

```yaml
name: set2-compose

services:
  web:
    build: .
    image: set2-web:1.0
    ports:
      - "8082:80"
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
docker network inspect set2-compose_app-net
docker compose exec db psql -U labuser -d labdb -c "SELECT current_database();"
```

Open **`http://localhost:8082`**. The page should display the Set 2 heading; the SQL command should return `labdb`.

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

Edit `index.html` in your local clone before the rebuild to observe a changed page. `restart` alone does not rebuild an image. To remove containers and the network while keeping records, run `docker compose down`.

**Troubleshooting:** check `docker compose ps`, service logs and `docker compose config`; inside the network use `db:5432` and `redis:6379`, never `localhost`. A shared network enables connectivity but does not itself enforce database authentication. If a host port is already occupied, choose a free host port and update the browser URL.

**Expected result:** all defined services run, the web page is reachable, SQL succeeds, and records survive container recreation.

### Scenario-based questions and one-line answers

1. **A developer is already running Nginx and PostgreSQL using Docker Compose. Now, the developer wants to add a Redis service. How can the existing Compose file be modified?**  
   Add a `redis` service using `image: redis:7-alpine`, attach it to the same network, then run `docker compose up -d`; add a `/data` volume if its data must persist.

2. **A web application requires a database to run properly. How can depends_on be used in Docker Compose to specify the startup dependency?**  
   Set `depends_on: {db: {condition: service_healthy}}` on the web service and define a database healthcheck; short-form `depends_on: [db]` only orders startup.

3. **An organization wants to run two instances of a web service to handle more requests. Which Docker Compose command can be used to scale the web service?**  
   Run `docker compose up -d --scale web=2` after replacing the single fixed host port with distinct/dynamic ports or a reverse proxy and avoiding a fixed `container_name`.

4. **A developer modifies the application code and wants to rebuild and restart the application using Docker Compose. Which command should be executed?**  
   Run `docker compose up -d --build` to rebuild image-based application code and recreate affected services.

5. **A developer wants to start, check, and stop multiple containers using Docker Compose. Which commands should be used?**  
   Use `docker compose up -d` to start, `docker compose ps` to check, and `docker compose stop` to stop the stack.

---

## Q2 Maven Jenkins Pipeline View and Poll SCM

### Question

> Q2. You are appointed as a DevOps engineer for automating the build process of a web application using Maven and to perform following tasks: [25 Marks]1. Pull the Maven web project from GitHub with link https://github.com/anujyog1/MetroRepo.git2. Build the project and create the Pipeline View using Jenkins.
>
> 3. Demonstrate use of POLL SCM.

**One-line explanation:** Build the Metro WAR in Jenkins, display its build flow and trigger builds only when Poll SCM detects a new revision.

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

#### Step 1 Create the Freestyle project

1. **Dashboard → New Item → Item name:** `Metro_Build` → **Freestyle project → OK**.
2. Select **JDK → JDK21** if the selector is shown.
3. **Source Code Management → Git → Repository URL:** `https://github.com/anujyog1/MetroRepo.git`.
4. **Credentials:** none for the public repository; use a stored Jenkins credential for an authorized private lab copy.
5. **Branches to build → Branch Specifier:** `*/master`.

#### Step 2 Configure Poll SCM

Under **Build Triggers**, tick **Poll SCM** and enter `H/5 * * * *`; leave **Build periodically** unticked. Jenkins checks Git about every five minutes and starts a build only when it detects a new revision. Click **Build Now** for the first baseline build. A push is detected at the next poll, not immediately. Do not change the original repository to force a demonstration; use an authorized lab copy for a controlled change.

#### Step 3 Add Maven and artifact actions

1. **Build Steps → Add build step → Invoke top-level Maven targets**.
2. **Maven Version:** `MAVEN-HOME`; **Goals:** `clean package`; POM at its default `pom.xml`.
3. **Post-build Actions → Archive the artifacts → Files to archive:** `target/*.war`; leave **Allow empty archive** unticked and archive only successful builds if the option is shown.
4. The inspected source has no test reports; do not add a mandatory empty JUnit report action unless you add real tests.
5. **Save → Build Now → build number → Console Output**. Look for the Git revision, Maven goals, **BUILD SUCCESS**, and **Finished: SUCCESS**.

`clean package` compiles sources, runs available tests, and creates the WAR through Maven’s lifecycle.

#### Step 4 Create Build Pipeline View

1. **Dashboard → + / New View** → name `Metro_Build_View` → **Build Pipeline View → Create**.
2. **Pipeline Flow → Select Initial Job → Initial Job:** `Metro_Build` → **Apply → Save**.
3. Open the view after building; it shows this Freestyle job’s build history and status. A single job appears as one node.

To show a linked flow, create a second Freestyle job named `Metro_Build_Verify` with **Copy from → Metro_Build**; untick its Poll SCM trigger, change its Maven goals to `clean verify`, and save. In **Metro_Build → Configure → Post-build Actions → Build other projects**, enter `Metro_Build_Verify` and choose **Trigger only if build is stable**. The view now shows Build → Verify; each job checks out its own workspace, and `verify` repeats earlier Maven phases.

**Troubleshooting:** wrong branch gives “couldn’t find any revision”; missing `pom.xml` means the wrong project directory; failed dependency downloads need internet/proxy configuration; read the first meaningful `[ERROR]` in Console Output. A missing artifact is not fixed by allowing an empty archive. Build Pipeline View supports Freestyle jobs; use Stage View for Groovy Pipeline jobs. [Build Pipeline documentation](https://plugins.jenkins.io/build-pipeline-plugin/)

---

## Q3 Nagios system and application monitoring

### Question

> Q3. A start-up company wants dashboard displaying real-time status and health of the system in the company. Demonstrate which monitoring tool you will use to monitor CPU usage, memory, and service uptime through dashboard. [25 Marks]

**One-line explanation:** Run Nagios Core with CPU, memory, disk, uptime and HTTP checks and observe the dashboard’s service states.

### Solution

#### Nagios Step 1 Start the monitoring server

In a separate working folder for Nagios, use Windows PowerShell:

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set2\nagios" | Out-Null
Set-Location "$HOME\se-lab-work\set2\nagios"
docker pull jasonrivers/nagios:latest
docker run -d --name set2-nagios --restart unless-stopped -p 8888:80 -e NAGIOS_TIMEZONE=Asia/Kolkata jasonrivers/nagios:latest
docker ps --filter name=set2-nagios
docker logs --tail 50 set2-nagios
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

Create **`lab.cfg`** in the same folder. The application must already be reachable on Windows host port **8082** at **`/`**:

```text
define command {
    command_name check_lab_http
    command_line $USER1$/check_http -H $HOSTADDRESS$ -p 8082 -u / -w 2 -c 5
}
define command {
    command_name check_lab_tcp
    command_line $USER1$/check_tcp -H $HOSTADDRESS$ -p 8082
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
docker cp .\check_lab_metrics.py set2-nagios:/opt/nagios/libexec/check_lab_metrics.py
docker cp .\lab.cfg set2-nagios:/opt/nagios/etc/conf.d/lab.cfg
docker exec set2-nagios /opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
```

Only after the preflight check reports **zero errors**, restart and test:

```powershell
docker restart set2-nagios
docker exec set2-nagios sh -c 'pgrep -a nagios'
docker exec set2-nagios /opt/nagios/libexec/check_http -H host.docker.internal -p 8082 -u / -w 2 -c 5
```

```powershell
docker exec set2-nagios python3 /opt/nagios/libexec/check_lab_metrics.py cpu
docker exec set2-nagios python3 /opt/nagios/libexec/check_lab_metrics.py memory
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

---

## Q4 Metro application on AWS EC2

### Question

> Q4. Your company wants you to deploy a application from github with link https://github.com/anujyog1/MetroRepo.git on an AWS EC2 Ubuntu instance. To show the deployment using public ip address, help your teammates to launch EC2, install docker and other required software’s, clone application, write the corresponding dockerfile, build the docker image and run it. [25 Marks]

**One-line explanation:** Build the Metro WAR in a Maven Docker stage and host it with Tomcat on Ubuntu EC2.

### Solution

#### Step 1 Launch and connect to Ubuntu EC2

1. **AWS Console → search EC2 → Instances → Launch instances**; choose your lab region and name `set2-app`.
2. **Application and OS Images → Ubuntu Server 24.04 LTS → 64-bit x86**. Choose the small instance allowed by your lab/account, for example `t3.micro`; use `t2.micro` if your paper/lab specifically requires it and it is offered. More memory may be needed for Maven builds. Free Tier eligibility depends on your account and selected resources; do not assume a machine is always free.
3. **Key pair → Create new key pair** → name `set2-key` → RSA → `.pem`; download and keep the key.
4. **Network settings → Edit:** use a public subnet with an Internet Gateway route; enable **Auto-assign public IP**; create a Security Group with **SSH TCP 22 → My IP**; add **Custom TCP 8080** for the application. For web ports, use **My IP** for your demo or allow public HTTP access when required by the question.
5. Launch; wait for **Running** and all status checks to pass. Select the instance → **Details → Public IPv4 address**; copy it.
6. **Connect → SSH client** shows the SSH command. In Windows PowerShell, switch to the folder containing the downloaded key and run:

```powershell
Set-Location "$HOME\Downloads"
ssh -i .\set2-key.pem ubuntu@PUBLIC_IP
```

Accept the new host key when prompted. If SSH times out, recheck port 22, your current My IP, public IP and subnet route; if it says `Permission denied (publickey)`, verify the key pair and username `ubuntu`. Inbound web access is a separate rule from SSH.

If Windows SSH specifically reports overly open private-key permissions, run these commands in the key folder, then retry SSH:

```powershell
$keyPath = (Resolve-Path '.\set2-key.pem').Path
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
git clone --branch master https://github.com/anujyog1/MetroRepo.git
cd MetroRepo
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
COPY --from=build /app/target/Book-Metro-Ticket.war /usr/local/tomcat/webapps/Book-Metro-Ticket.war
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

Maven produces `target/Book-Metro-Ticket.war`; the image copies it as `Book-Metro-Ticket.war`, making the URL context `/Book-Metro-Ticket/`. Tomcat 9 supports these repositories’ `javax.servlet`/JSP projects; do not substitute Tomcat 10/11 without a Jakarta migration. `EXPOSE` documents a port; `-p` in the next command publishes it.

#### Step 5 Build and run

```bash
sudo docker build -t set2-app:1.0 .
sudo docker run -d --name set2-app --restart unless-stopped -p 8080:8080 set2-app:1.0
sudo docker ps --filter name=set2-app
sudo docker logs --tail 100 set2-app
curl -I http://localhost:8080/Book-Metro-Ticket/
```

The final `.` in `docker build` is required. Wait for Tomcat to deploy the WAR; an HTTP success response from the local URL confirms application access.

#### Step 6 Verify from your laptop browser

Open **`http://PUBLIC_IP:8080/Book-Metro-Ticket/`**, replacing `PUBLIC_IP`. The path is case-sensitive. Check the home page and the supplied JSP links (`index.jsp`, `login.jsp`, `register.jsp`). Only demonstrate features actually implemented; a placeholder or client-side form is not a persistent database workflow.

#### Troubleshooting and reboot behaviour

- Local `curl` works but the browser times out: check inbound TCP 8080, the current public IP, public subnet routing, and `sudo ufw status` if Ubuntu’s firewall is enabled.
- HTTP 404: confirm `/Book-Metro-Ticket/`, inspect `sudo docker exec set2-app ls -l /usr/local/tomcat/webapps`, and read Tomcat logs.
- Container exited: run `sudo docker ps -a --filter name=set2-app` and `sudo docker logs set2-app`; do not assume Running EC2 means Running Tomcat.
- Maven errors: read the first `[ERROR]`; confirm `pom.xml`, packaging `war`, dependency connectivity and adequate memory. These JSP-only projects may report **No tests to run**.
- For a new image version, stop/remove only `set2-app`, rebuild, and rerun it. `sudo systemctl enable docker` and the restart policy restore the container after reboot unless explicitly stopped.
- A normal reboot usually retains the public IP; **stop/start** can change an auto-assigned public IP. Use an Elastic IP when a stable public IPv4 address is required.

**Expected result:** the fresh WAR runs on Tomcat and opens at `http://PUBLIC_IP:8080/Book-Metro-Ticket/`. After your demonstration, stop/terminate the dedicated lab instance when you are finished with it.

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

