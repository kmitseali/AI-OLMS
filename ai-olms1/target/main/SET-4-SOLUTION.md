# SET 4 Software Engineering Lab Internal II Solutions

**Original paper:** `SET-4.docx` in the MORNING folder.  
**Format:** Question → One-line explanation → Solution → Expected result; scenario questions receive one-line answers.

## Quick navigation

- [Before starting](#before-starting)
- [Q1 Tomcat and Nginx reverse proxy](#q1-tomcat-and-nginx-reverse-proxy)
- [Q2 Event application on AWS EC2](#q2-event-application-on-aws-ec2)
- [Q3 Jenkins Freestyle and Poll SCM](#q3-jenkins-freestyle-and-poll-scm)
- [Q4 Kubernetes scaling and Nagios](#q4-kubernetes-scaling-and-nagios)
- [Verification and sources](#verification-and-sources)

## Before starting

- **Local commands:** Windows PowerShell. **EC2 commands:** Ubuntu SSH terminal. Code blocks labelled YAML, Dockerfile, Java, Python, XML or Groovy are file contents unless explicitly described otherwise.
- Start Docker Desktop with its Linux engine. Check `docker version`, `docker compose version`, `git --version`, `java -version`, `mvn -version`, `kubectl version --client`, and `minikube version` as required by the question. Install Minikube using its [official guide](https://minikube.sigs.k8s.io/docs/start/) if missing, then reopen PowerShell.
- Keep practice files under `$HOME\se-lab-work\set4`; this answer file stays directly in the MORNING folder. Use a separate subfolder for each question.
- Jenkins is assumed to be installed and running: open [Jenkins](http://localhost:8080) and sign in with your existing account. Avoid mapping another local container to host port 8080 while Jenkins uses it; a question that explicitly requires 8080 explains how to handle the conflict.
- Replace placeholders such as `PUBLIC_IP`, `YOUR_EMAIL`, and `YOUR_NGROK_DOMAIN`. Use spaces in YAML, not tabs. Enable **File Explorer → View → Show → File name extensions** to avoid `.yml.txt` files.

### Repository checked for this paper

[Open the original GitHub repository](https://github.com/haleema91/LAB-INTERNAL-II-OEMS). Fresh read-only inspection on **20 November 2026** found:

| Item | Verified value |
|---|---|
| Branch | `main` |
| Inspected commit | `2697d7dcff35035efc54c5082f0ff950e8ade7e7` |
| Maven project root | Repository root, beside `pom.xml` |
| Build output | `target/OEMS-1.war` |
| Source tests | None in `src/test` |

**Important:** The actual WAR name is `OEMS-1.war`, not `OnlineEventManagement.war`; the deployment below deliberately renames it to match the question’s URL. The repository currently contains an index JSP, not the complete servlet-based booking system described in the scenario; its registration form targets `RegisterServlet`, which is absent and will not process registrations.

Repository inspection did not change GitHub. Webhook demonstrations below require a **lab repository you own or are authorized to administer**; use your own fork/copy and its URL rather than editing the question paper’s repository.

---

## Q1 Tomcat and Nginx reverse proxy

### Question

> Q1. Docker Compose – Online Event Management System Using Tomcat and NGINX [25 Marks]
>
> An organization wants to deploy an Online Event Management System using Docker Compose. The application is developed using Java (JSP and Servlets) and runs on an Apache Tomcat server. NGINX is used as a reverse proxy to handle incoming client requests and forward them to the Tomcat application server.
>
> The organization wants to use Docker Compose to manage both containers and establish communication between them.
>
> Tasks:
>
> Create a Docker Compose YAML file to run Apache Tomcat and NGINX in separate containers.
>
> Configure the Tomcat container to expose port 8080 and NGINX to expose port 80.
>
> Start both containers using the docker compose command.
>
> Verify the running containers, network, and services using Docker Compose commands.
>
> Access the Online Event Management application through the NGINX server using a web browser.

**One-line explanation:** Run the event WAR in Tomcat and forward browser requests to it through an Nginx reverse proxy.

### Solution

#### Step 1 Create the local working folder

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set4\q1" | Out-Null
Set-Location "$HOME\se-lab-work\set4\q1"
```

Create each named file in this folder using Notepad or VS Code, paste its code, and save it. In Notepad use **Save as type → All files** and keep the exact extension. File contents below are not terminal commands.

#### Step 2 Obtain the web application source

```powershell
git clone --branch main https://github.com/haleema91/LAB-INTERNAL-II-OEMS.git app
```

Create `Dockerfile` in the current question folder, not inside `app`, with these contents:

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY app/pom.xml ./
COPY app/src ./src
RUN mvn -B clean package

FROM tomcat:9.0-jdk21-temurin
COPY --from=build /app/target/OEMS-1.war /usr/local/tomcat/webapps/OnlineEventManagement.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

#### Step 3 Write nginx.conf

```nginx
server {
    listen 80;
    server_name _;
    location / {
        proxy_pass http://tomcat:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

`tomcat` is the Compose service name and 8080 is its container port. The proxy passes the original path through, so use `/OnlineEventManagement/` in the browser.

#### Step 4 Write docker-compose.yml

```yaml
name: set4-events
services:
  tomcat:
    build: .
    ports:
      - "8080:8080"
    networks: [event-net]
  nginx:
    image: nginx:stable
    depends_on: [tomcat]
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    networks: [event-net]
networks:
  event-net:
    driver: bridge
```

This question explicitly publishes host port 8080 for Tomcat. **Stop the local Jenkins service through Windows Services before running this stack**, or run this practical on a different lab machine. Start Jenkins again after `docker compose down` when moving to a Jenkins question. Do not leave both trying to bind the same host port.

#### Step 5 Start and inspect

```powershell
docker compose config --quiet
docker compose up -d --build
docker compose ps
docker compose logs --tail 60 tomcat
```

```powershell
docker compose logs --tail 40 nginx
docker network inspect set4-events_event-net
docker compose exec nginx nginx -t
docker compose exec nginx curl -I http://tomcat:8080/OnlineEventManagement/
```

Open **`http://localhost/OnlineEventManagement/`** through Nginx; compare direct Tomcat access at `http://localhost:8080/OnlineEventManagement/`. Short-form `depends_on` starts Tomcat first but does not wait for complete WAR deployment, so a brief initial 502 can occur; wait and check logs. For persistent 502, inspect the upstream service name, network membership, internal port and Tomcat startup.

When finished, use `docker compose stop` to pause or `docker compose down` to remove this stack. Keep named volumes by omitting `-v`. **Expected result:** the web app opens at its context URL, the containers share their custom network, and the requested port mappings are visible in `docker compose ps`.

### Scenario-based questions and one-line answers

1. **The NGINX container is unable to communicate with the Tomcat container. Explain how you will troubleshoot the issue by checking the custom Docker network, service names, container connectivity, and port configuration.**  
   Check `docker network inspect set4-events_event-net`, both service memberships, `tomcat:8080`, and `docker compose exec nginx curl -I http://tomcat:8080/OnlineEventManagement/`.

2. **The Tomcat container is running, but users receive a 502 Bad Gateway error when accessing the application through NGINX. Identify the possible causes and explain how you will troubleshoot the NGINX configuration, upstream server address, Tomcat status, and container logs.**  
   Validate `nginx -t`, verify upstream `tomcat:8080`, and inspect `docker compose logs nginx tomcat` for connection refusal, slow startup or a failed Tomcat process.

3. **Another application is already using port 80 on the host machine. Explain how you will modify the Docker Compose YAML file to expose NGINX on a different host port while keeping the internal NGINX port unchanged.**  
   Change Nginx’s mapping from `"80:80"` to `"8082:80"` and browse `http://localhost:8082/OnlineEventManagement/` while keeping Nginx’s internal listen port 80.

---

## Q2 Event application on AWS EC2

### Question

> Q2. AWS EC2 Deployment – Online Event Management Maven Web Application [25 Marks]
>
> An event management company has developed a Java-based Online Event Management web application using Maven, JSP, and Servlets. The application allows customers to view upcoming events, register for events, and book tickets.
>
> The administrator wants to deploy this Maven web application on an AWS EC2 instance using Apache Tomcat. The application must be packaged as a WAR file using Maven and deployed on the Tomcat server running on Ubuntu EC2.
>
> Task: Deploy the online event management application on AWS using an EC2 instance.( https://github.com/haleema91/LAB-INTERNAL-II-OEMS.git)

**One-line explanation:** Package the event WAR and deploy it as OnlineEventManagement on EC2 host port 9090.

### Solution

#### Step 1 Launch and connect to Ubuntu EC2

1. **AWS Console → search EC2 → Instances → Launch instances**; choose your lab region and name `set4-app`.
2. **Application and OS Images → Ubuntu Server 24.04 LTS → 64-bit x86**. Choose the small instance allowed by your lab/account, for example `t3.micro`; use `t2.micro` if your paper/lab specifically requires it and it is offered. More memory may be needed for Maven builds. Free Tier eligibility depends on your account and selected resources; do not assume a machine is always free.
3. **Key pair → Create new key pair** → name `set4-key` → RSA → `.pem`; download and keep the key.
4. **Network settings → Edit:** use a public subnet with an Internet Gateway route; enable **Auto-assign public IP**; create a Security Group with **SSH TCP 22 → My IP**; add **Custom TCP 9090** for the application. For web ports, use **My IP** for your demo or allow public HTTP access when required by the question.
5. Launch; wait for **Running** and all status checks to pass. Select the instance → **Details → Public IPv4 address**; copy it.
6. **Connect → SSH client** shows the SSH command. In Windows PowerShell, switch to the folder containing the downloaded key and run:

```powershell
Set-Location "$HOME\Downloads"
ssh -i .\set4-key.pem ubuntu@PUBLIC_IP
```

Accept the new host key when prompted. If SSH times out, recheck port 22, your current My IP, public IP and subnet route; if it says `Permission denied (publickey)`, verify the key pair and username `ubuntu`. Inbound web access is a separate rule from SSH.

If Windows SSH specifically reports overly open private-key permissions, run these commands in the key folder, then retry SSH:

```powershell
$keyPath = (Resolve-Path '.\set4-key.pem').Path
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
git clone --branch main https://github.com/haleema91/LAB-INTERNAL-II-OEMS.git
cd LAB-INTERNAL-II-OEMS
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
COPY --from=build /app/target/OEMS-1.war /usr/local/tomcat/webapps/OnlineEventManagement.war
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

Maven produces `target/OEMS-1.war`; the image copies it as `OnlineEventManagement.war`, making the URL context `/OnlineEventManagement/`. Tomcat 9 supports these repositories’ `javax.servlet`/JSP projects; do not substitute Tomcat 10/11 without a Jakarta migration. `EXPOSE` documents a port; `-p` in the next command publishes it.

#### Step 5 Build and run

```bash
sudo docker build -t set4-app:1.0 .
sudo docker run -d --name set4-app --restart unless-stopped -p 9090:8080 set4-app:1.0
sudo docker ps --filter name=set4-app
sudo docker logs --tail 100 set4-app
curl -I http://localhost:9090/OnlineEventManagement/
```

The final `.` in `docker build` is required. Wait for Tomcat to deploy the WAR; an HTTP success response from the local URL confirms application access.

#### Step 6 Verify from your laptop browser

Open **`http://PUBLIC_IP:9090/OnlineEventManagement/`**, replacing `PUBLIC_IP`. The path is case-sensitive. Check the home page and the supplied JSP links (`index.jsp`). Only demonstrate features actually implemented; a placeholder or client-side form is not a persistent database workflow.

#### Troubleshooting and reboot behaviour

- Local `curl` works but the browser times out: check inbound TCP 9090, the current public IP, public subnet routing, and `sudo ufw status` if Ubuntu’s firewall is enabled.
- HTTP 404: confirm `/OnlineEventManagement/`, inspect `sudo docker exec set4-app ls -l /usr/local/tomcat/webapps`, and read Tomcat logs.
- Container exited: run `sudo docker ps -a --filter name=set4-app` and `sudo docker logs set4-app`; do not assume Running EC2 means Running Tomcat.
- Maven errors: read the first `[ERROR]`; confirm `pom.xml`, packaging `war`, dependency connectivity and adequate memory. These JSP-only projects may report **No tests to run**.
- For a new image version, stop/remove only `set4-app`, rebuild, and rerun it. `sudo systemctl enable docker` and the restart policy restore the container after reboot unless explicitly stopped.
- A normal reboot usually retains the public IP; **stop/start** can change an auto-assigned public IP. Use an Elastic IP when a stable public IPv4 address is required.

**Expected result:** the fresh WAR runs on Tomcat and opens at `http://PUBLIC_IP:9090/OnlineEventManagement/`. After your demonstration, stop/terminate the dedicated lab instance when you are finished with it.

### Scenario-based questions and one-line answers

1. **The Tomcat server is running successfully on the EC2 instance, but users cannot access the Online Event Management application using the EC2 public IPv4 address. What changes are required in the EC2 Security Group to allow inbound traffic on port 9090? How will you verify the Tomcat port and troubleshoot connectivity issues?**  
   Allow inbound TCP 9090 from the required client source, verify the host mapping `9090:8080`, and compare local curl, Docker port output and Tomcat logs before checking external routing/firewalls.

2. **After deploying the WAR file, the administrator accesses the application using: http://<EC2-Public-IP>:9090/OnlineEventManagement/ The browser displays an HTTP 404 error. Identify the possible reasons and explain how you will verify WAR deployment, application naming, Tomcat logs, JSP files and servlet URL mappings.**  
   Confirm the WAR was deliberately named `OnlineEventManagement.war`, verify its webapps deployment and JSP/servlet mappings, and read Tomcat logs; the repository’s original context is `/OEMS-1/`.

3. **The administrator executes mvn clean package to generate the Online Event Management WAR file. However, the WAR file is not generated inside the target directory. Identify the possible causes and explain how you will troubleshoot the Maven project structure, pom.xml configuration, dependencies, and compilation errors.**  
   Run Maven from the folder with `pom.xml`, confirm `<packaging>war</packaging>`, and fix the first dependency/compilation/plugin error in `mvn clean package` output before inspecting `target/`.

---

## Q3 Jenkins Freestyle and Poll SCM

### Question

> Q3. Jenkins Freestyle Project – Maven Web Project [25 Marks]
>
> An event management company has developed a Java-based Online Event Management System using Maven, JSP, and Servlets. The source code is maintained in a GitHub repository. The company wants to automate the application build process using a Jenkins Freestyle project.
>
> The Jenkins server must fetch the source code from GitHub, execute Maven build commands, generate a WAR file, and verify the successful build.
>
> Tasks:
>
> Create a new Jenkins Freestyle Project named OnlineEventManagement.
>
> Clone the GitHub repository(https://github.com/haleema91/LAB-INTERNAL-II-OEMS.git)
>
> Configure the appropriate Git branch (main/master).
>
> Configure the build trigger using POLL scm to execute the project manually or whenever changes are pushed to GitHub.
>
> Verify the build execution status using the Jenkins pipeline view.

**One-line explanation:** Create the required Freestyle job, poll the source repository, generate a WAR and display build status in Pipeline View.

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

1. **Dashboard → New Item → Item name:** `OnlineEventManagement` → **Freestyle project → OK**.
2. Select **JDK → JDK21** if the selector is shown.
3. **Source Code Management → Git → Repository URL:** `https://github.com/haleema91/LAB-INTERNAL-II-OEMS.git`.
4. **Credentials:** none for the public repository; use a stored Jenkins credential for an authorized private lab copy.
5. **Branches to build → Branch Specifier:** `*/main`.

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

1. **Dashboard → + / New View** → name `OnlineEventManagement_View` → **Build Pipeline View → Create**.
2. **Pipeline Flow → Select Initial Job → Initial Job:** `OnlineEventManagement` → **Apply → Save**.
3. Open the view after building; it shows this Freestyle job’s build history and status. A single job appears as one node.

**Troubleshooting:** wrong branch gives “couldn’t find any revision”; missing `pom.xml` means the wrong project directory; failed dependency downloads need internet/proxy configuration; read the first meaningful `[ERROR]` in Console Output. A missing artifact is not fixed by allowing an empty archive. Build Pipeline View supports Freestyle jobs; use Stage View for Groovy Pipeline jobs. [Build Pipeline documentation](https://plugins.jenkins.io/build-pipeline-plugin/)

### Scenario-based questions and one-line answers

1. **The Jenkins Freestyle project is unable to fetch the Online Event Management source code from GitHub. Identify the possible causes and explain how you will troubleshoot the repository URL, Git installation, branch name, and GitHub credentials.**  
   Verify the repository URL, installed Git executable, actual `main` branch, network access and any required stored Jenkins credential using Console Output.

2. **The Jenkins build is successful in fetching the source code, but the WAR file is not generated in the target directory. Explain how you will troubleshoot the Maven packaging configuration, build goals, project structure, and Console Output.**  
   Check the workspace POM, `war` packaging, `clean package` goals, actual `target/OEMS-1.war` name and the first Maven error in Console Output.

3. **A developer modifies the event registration Servlet but introduces Java compilation errors. Jenkins reports the build as FAILED. Explain how you will identify the compilation errors using Console Output, correct the source code, and execute the build again.**  
   Read the compiler’s filename/line in Console Output, correct the source in an authorized lab copy or local demonstration, and rebuild; the inspected repository currently contains no event registration Servlet.

---

## Q4 Kubernetes scaling and Nagios

### Question

> Q4. Kubernetes – Application Deployment, Scaling and Monitoring [25 Marks]
>
> During popular events such as concerts, conferences, and festivals, thousands of users access the Online Event Management System simultaneously. The company wants to deploy multiple application replicas using Kubernetes to handle increased traffic and ensure application availability.Nagios will be used to monitor the application server.
>
> Tasks:
>
> Start a Kubernetes cluster using Minikube.
>
> Create a Kubernetes Deployment Nginx with two replicas.
>
> Expose the deployment using a NodePort Service and expose the application on port 6060.
>
> Scale the deployment from two to three replicas.
>
> Verify the deployment, Pods and Services using appropriate Kubernetes commands.
>
> Install and configure Nagios Core using Docker to monitor the application server.

**One-line explanation:** Deploy Nginx with two replicas, scale to three, expose port 6060 and monitor its endpoint with Nagios.

### Solution

#### Step 1 Create the local working folder

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set4\q4" | Out-Null
Set-Location "$HOME\se-lab-work\set4\q4"
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

The paper specifically requests an **Nginx Deployment**, so this practical uses the Nginx image rather than pretending that a generic Nginx page is the full business application. Start with 2 replicas and scale to 3.

#### Step 4 Create deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: set4-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: set4-nginx
  template:
    metadata:
      labels:
        app: set4-nginx
    spec:
      containers:
        - name: web
          image: nginx:stable
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: set4-nginx
spec:
  type: NodePort
  selector:
    app: set4-nginx
  ports:
    - port: 6060
      targetPort: 80
```

Save this file in the current working directory, then apply it:

```powershell
kubectl apply -f .\deployment.yaml
kubectl rollout status deployment/set4-nginx --timeout=180s
kubectl get deployments,pods,services
minikube service set4-nginx --url
```

The Service port is **6060**, while the container listens on **80**. Kubernetes allocates a NodePort, normally 30000–32767; **6060 is not the nodePort**. Open the URL printed by Minikube. On Windows with the Docker driver, keep the Minikube URL command running when it creates a tunnel.

#### Step 5 Access a fixed local port and allow Nagios to check it

In a separate PowerShell window, run this and leave it running:

```powershell
kubectl port-forward --address=0.0.0.0 service/set4-nginx 6060:6060
```

Open **`http://localhost:6060/`**. Binding to `0.0.0.0` makes the forwarding endpoint reachable from the Nagios Docker container through `host.docker.internal`; allow the relevant local firewall traffic if required. For a browser-only test without container access, omit `--address=0.0.0.0`.

#### Step 6 Scale and verify automatic recovery

```powershell
kubectl scale deployment/set4-nginx --replicas=3
kubectl rollout status deployment/set4-nginx --timeout=180s
kubectl get deployment set4-nginx
kubectl get pods -l app=set4-nginx -o wide
$podName = kubectl get pods -l app=set4-nginx -o jsonpath='{.items[0].metadata.name}'
kubectl delete pod $podName
kubectl get pods -l app=set4-nginx -w
```

Wait for **3/3** ready replicas. The ReplicaSet creates a replacement for a deleted Pod; press **Ctrl+C** after observing recovery. If a container merely crashes inside a still-existing Pod, kubelet normally restarts that container according to its restart policy; it does not always create a new Pod. Restart port forwarding if the selected Pod’s replacement interrupts the forwarding session.

#### Step 7 Inspect failures

```powershell
kubectl describe deployment set4-nginx
kubectl describe pods -l app=set4-nginx
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl logs deployment/set4-nginx --tail=60
kubectl get services
kubectl get endpointslices -l kubernetes.io/service-name=set4-nginx
```

For a restarting Pod, use `kubectl logs POD_NAME --previous` and `kubectl describe pod POD_NAME`. Pending Pods commonly lack CPU/memory capacity or have scheduling/volume constraints; an image pull problem normally shows `ImagePullBackOff`, not merely Pending forever. Check events before changing resources.

**Expected Kubernetes result:** browser access works, the Service routes to ready Pods, scaling reaches 3 replicas when resources permit, and a deleted Pod is replaced. The Nagios section below monitors the forwarded HTTP endpoint. [Minikube access](https://minikube.sigs.k8s.io/docs/handbook/accessing/)

#### Nagios Step 1 Start the monitoring server

In a separate working folder for Nagios, use Windows PowerShell:

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set4\nagios" | Out-Null
Set-Location "$HOME\se-lab-work\set4\nagios"
docker pull jasonrivers/nagios:latest
docker run -d --name set4-nagios --restart unless-stopped -p 8888:80 -e NAGIOS_TIMEZONE=Asia/Kolkata jasonrivers/nagios:latest
docker ps --filter name=set4-nagios
docker logs --tail 50 set4-nagios
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

Create **`lab.cfg`** in the same folder. The application must already be reachable on Windows host port **6060** at **`/`**:

```text
define command {
    command_name check_lab_http
    command_line $USER1$/check_http -H $HOSTADDRESS$ -p 6060 -u / -w 2 -c 5
}
define command {
    command_name check_lab_tcp
    command_line $USER1$/check_tcp -H $HOSTADDRESS$ -p 6060
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
docker cp .\check_lab_metrics.py set4-nagios:/opt/nagios/libexec/check_lab_metrics.py
docker cp .\lab.cfg set4-nagios:/opt/nagios/etc/conf.d/lab.cfg
docker exec set4-nagios /opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
```

Only after the preflight check reports **zero errors**, restart and test:

```powershell
docker restart set4-nagios
docker exec set4-nagios sh -c 'pgrep -a nagios'
docker exec set4-nagios /opt/nagios/libexec/check_http -H host.docker.internal -p 6060 -u / -w 2 -c 5
```

```powershell
docker exec set4-nagios python3 /opt/nagios/libexec/check_lab_metrics.py cpu
docker exec set4-nagios python3 /opt/nagios/libexec/check_lab_metrics.py memory
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

Stop port forwarding with Ctrl+C. Remove only this question’s resources with `kubectl delete service set4-nginx` and `kubectl delete deployment set4-nginx`. Stop monitoring with `docker stop set4-nagios` when finished; do not confuse a stopped monitoring server with an application recovery.

### Scenario-based questions and one-line answers

1. **During a major concert ticket release, the company increases the application replicas from two to five. Some Pods remain in the Pending state. What are the possible reasons and which Kubernetes commands will you use to troubleshoot the issue?**  
   Insufficient resources, scheduling constraints or unbound storage can prevent scheduling; inspect `kubectl describe pod POD`, `kubectl get events` and node capacity before adding capacity or correcting constraints.

2. **A customer cannot access the Online Event Management application through a browser even though the Pods are running. How will you configure and verify the NodePort Service and access the application through Minikube?**  
   Verify the Service selector and `targetPort: 80`, use Service port 6060 with an allocated valid NodePort, and open `minikube service set4-nginx --url` or forward `6060:6060`.

3. **One of the application Pods crashes during online ticket booking. Explain how Kubernetes Deployment ensures that the required number of replicas is maintained.**  
   Kubelet restarts a crashed container in its Pod; if the Pod is lost/deleted, the Deployment’s ReplicaSet creates a replacement to restore the desired replica count.

4. **The administrator wants to verify whether Nagios is running and monitoring the application server. Which commands will you execute to check Nagios status and configuration?**  
   Run `docker ps --filter name=set4-nagios`, `docker logs set4-nagios`, `docker exec set4-nagios pgrep -a nagios`, and the `/opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg` preflight check.

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

