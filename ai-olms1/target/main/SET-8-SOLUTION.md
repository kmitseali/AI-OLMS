# SET 8 Software Engineering Lab Internal II Solutions

**Original paper:** `SET-8.docx` in the MORNING folder.  
**Format:** Question → One-line explanation → Solution → Expected result; scenario questions receive one-line answers.

## Quick navigation

- [Before starting](#before-starting)
- [Q1 Vehicle Rental console application on AWS](#q1-vehicle-rental-console-application-on-aws)
- [Q2 Kubernetes Nginx scaling and Nagios](#q2-kubernetes-nginx-scaling-and-nagios)
- [Q3 Vehicle Rental Jenkins Freestyle and webhook](#q3-vehicle-rental-jenkins-freestyle-and-webhook)
- [Q4 WordPress and MySQL with Compose](#q4-wordpress-and-mysql-with-compose)
- [Verification and sources](#verification-and-sources)

## Before starting

- **Local commands:** Windows PowerShell. **EC2 commands:** Ubuntu SSH terminal. Code blocks labelled YAML, Dockerfile, Java, Python, XML or Groovy are file contents unless explicitly described otherwise.
- Start Docker Desktop with its Linux engine. Check `docker version`, `docker compose version`, `git --version`, `java -version`, `mvn -version`, `kubectl version --client`, and `minikube version` as required by the question. Install Minikube using its [official guide](https://minikube.sigs.k8s.io/docs/start/) if missing, then reopen PowerShell.
- Keep practice files under `$HOME\se-lab-work\set8`; this answer file stays directly in the MORNING folder. Use a separate subfolder for each question.
- Jenkins is assumed to be installed and running: open [Jenkins](http://localhost:8080) and sign in with your existing account. Avoid mapping another local container to host port 8080 while Jenkins uses it; a question that explicitly requires 8080 explains how to handle the conflict.
- Replace placeholders such as `PUBLIC_IP`, `YOUR_EMAIL`, and `YOUR_NGROK_DOMAIN`. Use spaces in YAML, not tabs. Enable **File Explorer → View → Show → File name extensions** to avoid `.yml.txt` files.

### Repository checked for this paper

[Open the original GitHub repository](https://github.com/haleema91/LAB-INTERNAL-II-OVR1). Fresh read-only inspection on **20 November 2026** found:

| Item | Verified value |
|---|---|
| Branch | `main` |
| Inspected commit | `96c6133dff60e390b3fd58277bb23cea4cba509e` |
| Maven project root | Repository root, beside `pom.xml` |
| Build output | `target/OVR-1-0.0.1-SNAPSHOT.jar` |
| Source tests | One trivial JUnit 5 test |

**Important:** Interactive Java 17 console program, main class `KMIT.OVR_1.App`; reads customer name, vehicle choice and days from standard input. The existing POM does not set a Main-Class manifest. Its single test is a trivial `assertTrue(true)` test, not booking validation.

Repository inspection did not change GitHub. Webhook demonstrations below require a **lab repository you own or are authorized to administer**; use your own fork/copy and its URL rather than editing the question paper’s repository.

---

## Q1 Vehicle Rental console application on AWS

### Question

> Q1. AWS EC2 Deployment – Online Vehicle Rental Maven Java Application [25 Marks]
>
> An online vehicle rental company has developed a Maven-based Java standalone application to manage vehicle rental details. The application allows administrators to add vehicle details, display available vehicles, calculate rental charges, and manage customer bookings.
>
> The company wants to deploy this Java standalone application on an AWS EC2 Ubuntu instance. The administrator must install git,Docker. Create a Docker image and execute the Java application inside a Docker container.
>
> Tasks:
>
> Launch an Ubuntu EC2 instance on AWS.
>
> Configure an appropriate instance type, key pair and Security Group.
>
> Clone the Online Vehicle Rental Maven Java standalone project from GitHub.( https://github.com/haleema91/LAB-INTERNAL-II-OVR1.git)
>
> Create a Dockerfile.
>
> Build the Docker image for the Maven Java standalone application.
>
> Run the Docker container and verify the application execution through container logs.

**One-line explanation:** Run the interactive Vehicle Rental Java program inside a Docker container on Ubuntu EC2 and verify its output/logs.

### Solution

#### Step 1 Launch and connect to Ubuntu EC2

1. **AWS Console → search EC2 → Instances → Launch instances**; choose your lab region and name `set8-app`.
2. **Application and OS Images → Ubuntu Server 24.04 LTS → 64-bit x86**. Choose the small instance allowed by your lab/account, for example `t3.micro`; use `t2.micro` if your paper/lab specifically requires it and it is offered. More memory may be needed for Maven builds. Free Tier eligibility depends on your account and selected resources; do not assume a machine is always free.
3. **Key pair → Create new key pair** → name `set8-key` → RSA → `.pem`; download and keep the key.
4. **Network settings → Edit:** use a public subnet with an Internet Gateway route; enable **Auto-assign public IP**; create a Security Group with **SSH TCP 22 → My IP**. For web ports, use **My IP** for your demo or allow public HTTP access when required by the question.
5. Launch; wait for **Running** and all status checks to pass. Select the instance → **Details → Public IPv4 address**; copy it.
6. **Connect → SSH client** shows the SSH command. In Windows PowerShell, switch to the folder containing the downloaded key and run:

```powershell
Set-Location "$HOME\Downloads"
ssh -i .\set8-key.pem ubuntu@PUBLIC_IP
```

Accept the new host key when prompted. If SSH times out, recheck port 22, your current My IP, public IP and subnet route; if it says `Permission denied (publickey)`, verify the key pair and username `ubuntu`. Inbound web access is a separate rule from SSH.

If Windows SSH specifically reports overly open private-key permissions, run these commands in the key folder, then retry SSH:

```powershell
$keyPath = (Resolve-Path '.\set8-key.pem').Path
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

This is a console application, so **only inbound SSH 22 is needed for this question**; do not open HTTP ports for the console program. There is no browser application URL or Tomcat requirement here.

#### Step 3 Clone the repository and create its Dockerfile

```bash
cd ~
git clone --branch main https://github.com/haleema91/LAB-INTERNAL-II-OVR1.git
cd LAB-INTERNAL-II-OVR1
nano Dockerfile
```

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml ./
COPY src ./src
RUN mvn -B clean package
RUN jar --update --file target/OVR-1-0.0.1-SNAPSHOT.jar --main-class KMIT.OVR_1.App

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/OVR-1-0.0.1-SNAPSHOT.jar /app/rental.jar
ENTRYPOINT ["java", "-jar", "/app/rental.jar"]
```

The POM builds `OVR-1-0.0.1-SNAPSHOT.jar` but does not give it a Main-Class manifest. The `jar --update ... --main-class` command adds that entry to the newly built local JAR, allowing `java -jar` in the runtime image; GitHub is unchanged. The program has no external runtime library dependency, so a plain JAR is sufficient here.

#### Step 4 Build and run interactively

```bash
sudo docker build -t set8-rental:1.0 .
sudo docker run -it --name set8-rental set8-rental:1.0
```

At the prompts, enter **Asha**, vehicle **1**, and **2** rental days. The program should show Toyota Innova, total **5000.0**, and booking status **CONFIRMED**, then finish. `-i` keeps standard input available; `-t` provides an interactive terminal. Running it detached with closed stdin can cause a Scanner input error.

#### Step 5 Inspect execution and container exit

```bash
sudo docker ps -a --filter name=set8-rental
sudo docker logs set8-rental
sudo docker inspect set8-rental --format '{{.State.ExitCode}}'
```

A successful short-lived console program normally shows **Exited (0)**; it is not supposed to remain a web server. To rerun the same container interactively, use `sudo docker start -ai set8-rental`. To demonstrate stopping a container that is waiting for input, run `sudo docker run -dit --name set8-wait set8-rental:1.0`, inspect its status, then `sudo docker stop set8-wait`. Do not apply a restart-always policy to a completed console task.

#### Optional deterministic log demonstration

```bash
printf 'Asha\n1\n2\n' | sudo docker run -i --name set8-rental-batch set8-rental:1.0
sudo docker logs set8-rental-batch
```

To build outside Docker when Maven is installed, `mvn clean package` already covers compilation and available tests; run `java -cp target/OVR-1-0.0.1-SNAPSHOT.jar KMIT.OVR_1.App` for the original JAR without a manifest. The included `assertTrue(true)` test does not validate rental calculations. **Expected result:** execution and booking output are visible in the terminal and saved container logs, followed by exit code 0. Terminate the dedicated lab EC2 instance when finished.

### Scenario-based questions and one-line answers

1. **The Online Vehicle Rental administrator has successfully launched an Ubuntu EC2 instance but needs to access it remotely. How will you connect to the EC2 instance from your local machine using an SSH key pair? What steps will you follow if the SSH connection fails?**  
   Connect with `ssh -i KEY.pem ubuntu@PUBLIC_IP`; troubleshoot key/username, key-file permissions, running status, current public IP, SSH/My IP rule and public routing.

2. **The development team has uploaded the Online Vehicle Rental Maven Java standalone project to GitHub. The administrator needs to clone the repository and generate the executable JAR file. Which Maven commands will you execute to clean previous build files, compile the Java source code, execute test cases, and package the application into a JAR file?**  
   After cloning the repository, run `mvn clean compile`, `mvn test`, and `mvn package` (or simply `mvn clean package`); add a Main-Class manifest or use `java -cp` for this original JAR.

3. **The Docker image has been created successfully, but the administrator wants to verify whether the Online Vehicle Rental Java application is executing properly inside the container. Which Docker commands will you use to run the container, check its status, inspect application logs, and stop the container?**  
   Use `docker run -it --name rental IMAGE`, `docker ps -a`, `docker logs rental` and `docker stop rental`; normal completion of this console task produces Exited (0).

---

## Q2 Kubernetes Nginx scaling and Nagios

### Question

> Q2. Kubernetes – Container Deployment, Auto Scaling and Monitoring – Online Vehicle rental [25 Marks]
>
> An automobile service management company has developed a Maven-based Java application to manage vehicle servicing, customer information, service appointments, and maintenance records.
>
> During festive seasons and promotional service campaigns, a large number of customers access the application simultaneously to schedule vehicle servicing, check service status and view maintenance details.
>
> The company wants to deploy its Java application using Kubernetes to handle increased customer traffic, maintain application availability through automatic Pod recovery and monitor application availability using Nagios.
>
> Tasks:
>
> Start a Kubernetes cluster using Minikube.
>
> Create a Kubernetes Deployment nginx with three replicas.
>
> Expose the deployment using a Kubernetes Service of type NodePort at port 6060.
>
> Scale the deployment from three to five replicas to handle increased customer traffic.
>
> Verify deployments, Pods and Services using appropriate Kubernetes commands.
>
> Install and configure Nagios Core using Docker to monitor the Online Vehicle Service Management application.

**One-line explanation:** Run three Nginx replicas, scale to five, expose port 6060 and monitor service availability through Nagios.

### Solution

#### Step 1 Create the local working folder

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set8\q2" | Out-Null
Set-Location "$HOME\se-lab-work\set8\q2"
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

The paper specifically requests an **Nginx Deployment**, so this practical uses the Nginx image rather than pretending that a generic Nginx page is the full business application. Start with 3 replicas and scale to 5.

#### Step 4 Create deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: set8-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: set8-nginx
  template:
    metadata:
      labels:
        app: set8-nginx
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
  name: set8-nginx
spec:
  type: NodePort
  selector:
    app: set8-nginx
  ports:
    - port: 6060
      targetPort: 80
```

Save this file in the current working directory, then apply it:

```powershell
kubectl apply -f .\deployment.yaml
kubectl rollout status deployment/set8-nginx --timeout=180s
kubectl get deployments,pods,services
minikube service set8-nginx --url
```

The Service port is **6060**, while the container listens on **80**. Kubernetes allocates a NodePort, normally 30000–32767; **6060 is not the nodePort**. Open the URL printed by Minikube. On Windows with the Docker driver, keep the Minikube URL command running when it creates a tunnel.

#### Step 5 Access a fixed local port and allow Nagios to check it

In a separate PowerShell window, run this and leave it running:

```powershell
kubectl port-forward --address=0.0.0.0 service/set8-nginx 6060:6060
```

Open **`http://localhost:6060/`**. Binding to `0.0.0.0` makes the forwarding endpoint reachable from the Nagios Docker container through `host.docker.internal`; allow the relevant local firewall traffic if required. For a browser-only test without container access, omit `--address=0.0.0.0`.

#### Step 6 Scale and verify automatic recovery

```powershell
kubectl scale deployment/set8-nginx --replicas=5
kubectl rollout status deployment/set8-nginx --timeout=180s
kubectl get deployment set8-nginx
kubectl get pods -l app=set8-nginx -o wide
$podName = kubectl get pods -l app=set8-nginx -o jsonpath='{.items[0].metadata.name}'
kubectl delete pod $podName
kubectl get pods -l app=set8-nginx -w
```

Wait for **5/5** ready replicas. The ReplicaSet creates a replacement for a deleted Pod; press **Ctrl+C** after observing recovery. If a container merely crashes inside a still-existing Pod, kubelet normally restarts that container according to its restart policy; it does not always create a new Pod. Restart port forwarding if the selected Pod’s replacement interrupts the forwarding session.

#### Step 7 Inspect failures

```powershell
kubectl describe deployment set8-nginx
kubectl describe pods -l app=set8-nginx
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl logs deployment/set8-nginx --tail=60
kubectl get services
kubectl get endpointslices -l kubernetes.io/service-name=set8-nginx
```

For a restarting Pod, use `kubectl logs POD_NAME --previous` and `kubectl describe pod POD_NAME`. Pending Pods commonly lack CPU/memory capacity or have scheduling/volume constraints; an image pull problem normally shows `ImagePullBackOff`, not merely Pending forever. Check events before changing resources.

#### Optional extension Demonstrate real CPU autoscaling

The scale command above is **manual scaling**. To demonstrate an HPA, enable metrics and use the CPU requests already present in the YAML:

```powershell
minikube addons enable metrics-server
kubectl top pods
kubectl autoscale deployment set8-nginx --cpu-percent=60 --min=3 --max=5
kubectl get hpa -w
```

Wait for valid metrics; no load means the HPA may stay at its minimum. It will not necessarily reach 5 replicas without sustained CPU use. Press Ctrl+C after observing it; remove this optional HPA with `kubectl delete hpa set8-nginx` before expecting later manual replica counts to remain fixed. [Kubernetes HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)

**Expected Kubernetes result:** browser access works, the Service routes to ready Pods, scaling reaches 5 replicas when resources permit, and a deleted Pod is replaced. The Nagios section below monitors the forwarded HTTP endpoint. [Minikube access](https://minikube.sigs.k8s.io/docs/handbook/accessing/)

#### Nagios Step 1 Start the monitoring server

In a separate working folder for Nagios, use Windows PowerShell:

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set8\nagios" | Out-Null
Set-Location "$HOME\se-lab-work\set8\nagios"
docker pull jasonrivers/nagios:latest
docker run -d --name set8-nagios --restart unless-stopped -p 8888:80 -e NAGIOS_TIMEZONE=Asia/Kolkata jasonrivers/nagios:latest
docker ps --filter name=set8-nagios
docker logs --tail 50 set8-nagios
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
docker cp .\check_lab_metrics.py set8-nagios:/opt/nagios/libexec/check_lab_metrics.py
docker cp .\lab.cfg set8-nagios:/opt/nagios/etc/conf.d/lab.cfg
docker exec set8-nagios /opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
```

Only after the preflight check reports **zero errors**, restart and test:

```powershell
docker restart set8-nagios
docker exec set8-nagios sh -c 'pgrep -a nagios'
docker exec set8-nagios /opt/nagios/libexec/check_http -H host.docker.internal -p 6060 -u / -w 2 -c 5
```

```powershell
docker exec set8-nagios python3 /opt/nagios/libexec/check_lab_metrics.py cpu
docker exec set8-nagios python3 /opt/nagios/libexec/check_lab_metrics.py memory
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

Stop port forwarding with Ctrl+C. Remove only this question’s resources with `kubectl delete service set8-nginx` and `kubectl delete deployment set8-nginx`. Stop monitoring with `docker stop set8-nagios` when finished; do not confuse a stopped monitoring server with an application recovery.

### Scenario-based questions and one-line answers

1. **The Online Vehicle Service Management application has been deployed successfully inside Minikube, but customers are unable to access the application through a web browser. How will you expose the application externally using a NodePort Service and access it through the Minikube service URL? What configurations are required in the Docker image and Kubernetes Service?**  
   Configure Service port 6060 with `targetPort: 80` for Nginx, keep labels/selectors matching, and use `minikube service set8-nginx --url` or port forwarding; a console JAR itself is not an HTTP server.

2. **During a special vehicle servicing campaign, the administrator scales the Online Vehicle Service Management deployment from two to four replicas. However, some Pods remain in the Pending state. What could be the possible reasons, and how would you troubleshoot the issue using Kubernetes commands?**  
   Inspect resources, node scheduling/taint rules and unbound storage with `kubectl describe pod`, `kubectl describe node` and sorted `kubectl get events`.

3. **The company administrator wants to monitor the availability of the Online Vehicle Service Management application deployed in Kubernetes. How will you install and configure Nagios Core using Docker? Which commands will you execute to verify Nagios service status and application monitoring?**  
   Run the Nagios image on 8888, load a valid HTTP host/service config under `/opt/nagios/etc/conf.d`, validate it, restart and check container/daemon status plus the Services dashboard.

---

## Q3 Vehicle Rental Jenkins Freestyle and webhook

### Question

> Q3. Jenkins Freestyle Project – Online Vehicle Rental Maven Java Application [25 Marks]
>
> An online vehicle rental company frequently updates its Java standalone application with new vehicle details, rental charges, customer registration, and booking functionalities. The development team maintains the Maven Java project source code in a GitHub repository.
>
> The company wants to automate the application build process using a Jenkins Freestyle Project. Configure Jenkins to retrieve the source code from GitHub, execute Maven build operations, generate an executable JAR file, and run the application.
>
> Tasks:
>
> Create a new Freestyle Project in Jenkins named Vehicle-Rental.
>
> Clone the GitHub repository. https://github.com/haleema91/LAB-INTERNAL-II-OVR1.git
>
> Execute the following Maven build operations:
>
> Clean,install,test
>
> Execute the Jenkins build and verify the Console Output and show pipeline view.
>
> Configure GitHub Webhooks to trigger the Jenkins Freestyle Project automatically whenever developers push code changes.

**One-line explanation:** Use the Vehicle-Rental Freestyle job to build, execute and archive the JAR and receive webhook-triggered builds.

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

1. **Dashboard → New Item → Item name:** `Vehicle-Rental` → **Freestyle project → OK**.
2. Select **JDK → JDK21** if the selector is shown.
3. **Source Code Management → Git → Repository URL:** `https://github.com/haleema91/LAB-INTERNAL-II-OVR1.git`.
4. **Credentials:** none for the public repository; use a stored Jenkins credential for an authorized private lab copy.
5. **Branches to build → Branch Specifier:** `*/main`.

#### Step 2 Select the webhook trigger

Under **Build Triggers**, tick **GitHub hook trigger for GITScm polling**. Leave timed **Poll SCM** and **Build periodically** unticked. Complete the webhook instructions below after saving the job.

#### Step 3 Add Maven and artifact actions

1. **Build Steps → Add build step → Invoke top-level Maven targets**.
2. **Maven Version:** `MAVEN-HOME`; **Goals:** `clean install test`; POM at its default `pom.xml`.
3. **Post-build Actions → Archive the artifacts → Files to archive:** `target/*.jar`; leave **Allow empty archive** unticked and archive only successful builds if the option is shown.
4. Add **Publish JUnit test result report** with `target/surefire-reports/*.xml`.
5. **Save → Build Now → build number → Console Output**. Look for the Git revision, Maven goals, **BUILD SUCCESS**, and **Finished: SUCCESS**.

`clean install test` intentionally repeats tests after `install` to match the paper; `install` already includes test and package.

#### Step 4 Run the console application without hanging Jenkins

Before the post-build actions, add **Build Steps → Execute Windows batch command** after Maven:

```bat
@echo off
jar --update --file target\OVR-1-0.0.1-SNAPSHOT.jar --main-class KMIT.OVR_1.App
if errorlevel 1 exit /b 1
(echo Demo Customer&echo 1&echo 2) > rental-input.txt
java -jar target\OVR-1-0.0.1-SNAPSHOT.jar < rental-input.txt
```

The input supplies a customer name, vehicle 1, and two rental days. Console Output should show **Toyota Innova**, total **5000.0**, and **CONFIRMED**. The `jar` command adds the missing Main-Class manifest to the generated target JAR before execution and archival; the original repository and the already installed Maven-cache copy are unchanged. Ensure the configured JDK’s `bin` directory is on this build step’s PATH so both `jar` and `java` resolve.

#### Step 5 Create Build Pipeline View

1. **Dashboard → + / New View** → name `Vehicle-Rental_View` → **Build Pipeline View → Create**.
2. **Pipeline Flow → Select Initial Job → Initial Job:** `Vehicle-Rental` → **Apply → Save**.
3. Open the view after building; it shows this Freestyle job’s build history and status. A single job appears as one node.

**Troubleshooting:** wrong branch gives “couldn’t find any revision”; missing `pom.xml` means the wrong project directory; failed dependency downloads need internet/proxy configuration; read the first meaningful `[ERROR]` in Console Output. A missing artifact is not fixed by allowing an empty archive. Build Pipeline View supports Freestyle jobs; use Stage View for Groovy Pipeline jobs. [Build Pipeline documentation](https://plugins.jenkins.io/build-pipeline-plugin/)

#### Configure the GitHub webhook without timed polling

These are instructions for **your own authorized lab repository**. No webhook, fork, commit or push has been created by this answer. If you cannot administer a repository, its owner must perform the GitHub-side steps; a webhook to someone else’s repository cannot be configured from Jenkins alone.

1. For a reproducible push demo, use a personal fork/lab copy of the question’s project; update the Jenkins checkout URL to that copy, keeping the correct branch.
2. In **Vehicle-Rental → Configure → Build Triggers**, tick **GitHub hook trigger for GITScm polling**. Untick scheduled **Poll SCM** and **Build periodically** → **Save**. The trigger’s name includes “polling”, but the poll is initiated by the webhook rather than a cron schedule.
3. Download/install the Windows agent from [ngrok](https://ngrok.com/download), sign in to the ngrok dashboard, copy your agent authtoken, and run `ngrok config add-authtoken YOUR_TOKEN` in PowerShell once. Then run `ngrok http 8080`; use the HTTPS forwarding domain it prints and leave that terminal running. Keep the actual token out of screenshots and source files.
4. In **your lab GitHub repository → Settings → Webhooks → Add webhook**, set **Payload URL:** `https://YOUR_NGROK_DOMAIN/github-webhook/` including the trailing slash; **Content type:** `application/json`; choose **Just the push event**, tick **Active**, and save. If you configure a webhook secret, configure the matching verification secret in your Jenkins GitHub integration.
5. Run one manual Jenkins build to establish the checkout/repository association. In **GitHub → Webhooks → Recent Deliveries**, verify a successful response, and confirm that ngrok shows the incoming request.
6. In your lab copy only, change a harmless line such as README text on the configured branch, commit and push. Do not click **Build Now** for this second run.
7. Confirm a new Jenkins build appears with a GitHub push cause, Console Output shows the new revision, and the appropriate Pipeline/Build Pipeline view updates.
8. Capture the webhook configuration, successful delivery, Jenkins build cause, and Console Output. Do not expose tokens or credentials in screenshots.

**If it does not trigger:** check ngrok is still forwarding to local port 8080, the current HTTPS domain matches the webhook, the endpoint ends `/github-webhook/`, Jenkins is reachable, and the repository/branch matches. A restarted free ngrok tunnel may have a different address. `localhost` cannot be reached directly by GitHub. An already public Jenkins server does not need ngrok. [Jenkins GitHub plugin](https://plugins.jenkins.io/github/)

### Scenario-based questions and one-line answers

1. **The Jenkins build has completed successfully and generated the executable JAR file inside the target directory. How will you configure the Post-Build Action to archive the generated JAR file? Which file pattern should you specify?**  
   Add Post-build Action → Archive the artifacts with `target/*.jar`, keep empty archives disallowed, and archive successful builds.

2. **Developers have added new vehicle registration and booking features to GitHub. The company wants Jenkins to automatically execute the Freestyle Project whenever developers push changes. How will you configure the GitHub webhook and the appropriate build trigger in Jenkins?**  
   In an authorized repository add a push webhook ending `/github-webhook/`, and tick Jenkins’s GitHub hook trigger for GITScm polling with no timed polling schedule.

3. **The Jenkins Freestyle Project fails during Maven execution, displaying errors related to Java or Maven configuration. How will you identify the issue, verify the configured JDK and Maven versions, and troubleshoot the failed build using Console Output?**  
   Read the first error in Console Output, verify Jenkins JDK/Maven tool names and paths, run version checks on the agent and confirm Java 17-or-newer compilation support.

---

## Q4 WordPress and MySQL with Compose

### Question

> Q4. Docker Compose – Online Vehicle Rental Website Using WordPress and MySQL [25 Marks]
>
> An online vehicle rental company wants to develop a website where customers can view available vehicles, rental prices, and booking information. The company wants to use WordPress for website development and MySQL for database management.
>
> Create a Docker Compose file to run WordPress and MySQL in separate containers.
>
> Tasks:
>
> Create a Docker Compose YAML file.
>
> Configure WordPress and MySQL containers.
>
> Configure port mappings and persistent volumes.
>
> Configure environment variables for database connectivity.
>
> Start both containers using docker compose.
>
> Verify running containers using Docker Compose commands.
>
> Access the WordPress website through a web browser.

**One-line explanation:** Run a WordPress vehicle website and MySQL with matching credentials, published HTTP access and persistent volumes.

### Solution

#### Step 1 Create the local working folder

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set8\q4" | Out-Null
Set-Location "$HOME\se-lab-work\set8\q4"
```

Create each named file in this folder using Notepad or VS Code, paste its code, and save it. In Notepad use **Save as type → All files** and keep the exact extension. File contents below are not terminal commands.

#### Step 2 Write docker-compose.yml

```yaml
name: set8-wordpress
services:
  db:
    image: mysql:8.4
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: LabRootPass
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: LabWpPass
    volumes:
      - mysql-data:/var/lib/mysql
    networks: [wp-net]
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -p$$MYSQL_ROOT_PASSWORD --silent"]
      interval: 10s
      timeout: 5s
      retries: 15
      start_period: 30s
  wordpress:
    image: wordpress:php8.2-apache
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "8088:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: LabWpPass
    volumes:
      - wordpress-data:/var/www/html
    networks: [wp-net]
volumes:
  mysql-data:
  wordpress-data:
networks:
  wp-net:
    driver: bridge
```

MySQL stays on the internal Compose network; WordPress connects using `db:3306`, not `localhost`. Matching environment credentials establish the database connection. These passwords are lab examples.

#### Step 3 Start the website

```powershell
docker compose config --quiet
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail 40 db wordpress
```

Wait for the database to become healthy. Open **`http://localhost:8088`**, select a language, enter **Site Title**, administrator username/password and email, then **Install WordPress → Log In**. Add a test post from **Posts → Add New Post** and publish it.

#### Step 4 Confirm persistence

```powershell
docker compose down
docker compose up -d
docker compose ps
```

Wait for startup and reopen the site; the test post and setup should remain. Do not use `down -v` when retaining data. WordPress users/posts are in MySQL; uploaded files/themes/plugins are in the WordPress volume. If connectivity fails, inspect logs and compare the database name, user, password and `db:3306` setting. **Expected result:** a working vehicle-rental WordPress website with records preserved after container recreation; its content is separate from the Java console app in Q1.

### Scenario-based questions and one-line answers

1. **You are running the Online Vehicle Rental application using Java and MySQL containers. After removing and recreating the MySQL container, you notice that all vehicle details, customer registrations, and booking information are lost. Why does this happen, and how can you configure Docker Compose volumes to preserve database data?**  
   The data was in the deleted container layer; declare a named volume and mount it at `/var/lib/mysql`, retaining the volume across container recreation.

2. **Your Online Vehicle Rental Java application is deployed using Docker Compose. However, the application fails to connect to the MySQL database. How do you identify whether both containers are running, check their logs.**  
   Use `docker compose ps` and `docker compose logs web db` (or `wordpress db` for this stack), then verify shared network, service-name host and matching database credentials.

3. **Your Maven Java standalone application is successfully packaged into a JAR file, but the Docker container exits immediately after execution. How will you identify the issue using Docker commands, inspect container logs and troubleshoot  application execution errors?**  
   Inspect `docker ps -a`, logs and exit code: exit 0 may mean normal task completion, while missing stdin, an absent main manifest or Java exceptions need the appropriate input/entry-point fix.

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

