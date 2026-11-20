# SET 6 Software Engineering Lab Internal II Solutions

**Original paper:** `SET-6.docx` in the MORNING folder.  
**Format:** Question → One-line explanation → Solution → Expected result; scenario questions receive one-line answers.

## Quick navigation

- [Before starting](#before-starting)
- [Q1 Tomcat and MySQL with Compose](#q1-tomcat-and-mysql-with-compose)
- [Q2 Jenkins webhooks and Tomcat deployment](#q2-jenkins-webhooks-and-tomcat-deployment)
- [Q3 Kubernetes scaling and Nagios](#q3-kubernetes-scaling-and-nagios)
- [Q4 Static Nginx website on AWS EC2](#q4-static-nginx-website-on-aws-ec2)
- [Verification and sources](#verification-and-sources)

## Before starting

- **Local commands:** Windows PowerShell. **EC2 commands:** Ubuntu SSH terminal. Code blocks labelled YAML, Dockerfile, Java, Python, XML or Groovy are file contents unless explicitly described otherwise.
- Start Docker Desktop with its Linux engine. Check `docker version`, `docker compose version`, `git --version`, `java -version`, `mvn -version`, `kubectl version --client`, and `minikube version` as required by the question. Install Minikube using its [official guide](https://minikube.sigs.k8s.io/docs/start/) if missing, then reopen PowerShell.
- Keep practice files under `$HOME\se-lab-work\set6`; this answer file stays directly in the MORNING folder. Use a separate subfolder for each question.
- Jenkins is assumed to be installed and running: open [Jenkins](http://localhost:8080) and sign in with your existing account. Avoid mapping another local container to host port 8080 while Jenkins uses it; a question that explicitly requires 8080 explains how to handle the conflict.
- Replace placeholders such as `PUBLIC_IP`, `YOUR_EMAIL`, and `YOUR_NGROK_DOMAIN`. Use spaces in YAML, not tabs. Enable **File Explorer → View → Show → File name extensions** to avoid `.yml.txt` files.

### Repository checked for this paper

[Open the original GitHub repository](https://github.com/haleema91/LAB-INTERNAL-II-OBS). Fresh read-only inspection on **20 November 2026** found:

| Item | Verified value |
|---|---|
| Branch | `main` |
| Inspected commit | `c4cc3cd038a25e4cc9150177705be569415197f2` |
| Maven project root | Repository root, beside `pom.xml` |
| Build output | `target/OBS-1.war` |
| Source tests | None in `src/test` |

**Important:** WAR with a JSP boutique page. No Java unit tests or implemented MySQL-backed checkout are included. Q4 separately asks for a static Nginx website, so it does not require deploying this WAR.

Repository inspection did not change GitHub. Webhook demonstrations below require a **lab repository you own or are authorized to administer**; use your own fork/copy and its URL rather than editing the question paper’s repository.

---

## Q1 Tomcat and MySQL with Compose

### Question

> Q1.Docker Compose – Online Boutique Store (25 Marks)
>
> An online boutique store wants to launch an e-commerce website. The application consists of a mysql backend and an tomcat web server. Create a Docker Compose file to run both applications in separate containers, exposing the mysql backend on port 3306 and tomcat on port 8080.
>
> Tasks:
>
> Create a Docker Compose YAML file.
>
> Configure appropriate port mappings and persistent volumes.
>
> Start both containers using docker compose
>
> Verify running containers using docker compose command.
>
> Access the tomcat server through a web browser.

**One-line explanation:** Run the boutique WAR on Tomcat with a persistent MySQL container and the requested host ports.

### Solution

#### Step 1 Create the local working folder

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set6\q1" | Out-Null
Set-Location "$HOME\se-lab-work\set6\q1"
```

Create each named file in this folder using Notepad or VS Code, paste its code, and save it. In Notepad use **Save as type → All files** and keep the exact extension. File contents below are not terminal commands.

#### Step 2 Obtain the web application source

```powershell
git clone --branch main https://github.com/haleema91/LAB-INTERNAL-II-OBS.git app
```

Create `Dockerfile` in the current question folder, not inside `app`, with these contents:

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY app/pom.xml ./
COPY app/src ./src
RUN mvn -B clean package

FROM tomcat:9.0-jdk21-temurin
COPY --from=build /app/target/OBS-1.war /usr/local/tomcat/webapps/OBS-1.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

#### Step 4 Write docker-compose.yml

```yaml
name: set6-boutique
services:
  db:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD: LabRootPass
      MYSQL_DATABASE: boutique
      MYSQL_USER: boutiqueuser
      MYSQL_PASSWORD: LabBoutiquePass
    ports:
      - "127.0.0.1:3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    networks: [boutique-net]
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -p$$MYSQL_ROOT_PASSWORD --silent"]
      interval: 10s
      timeout: 5s
      retries: 15
      start_period: 30s
  tomcat:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy
    networks: [boutique-net]
volumes:
  mysql-data:
networks:
  boutique-net:
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
docker network inspect set6-boutique_boutique-net
docker compose exec db mysql -uboutiqueuser -p boutique
```

At the MySQL password prompt, enter `LabBoutiquePass`; run `SHOW DATABASES;`, then `exit`. Open **`http://localhost:8080/OBS-1/`**; the official Tomcat image may return 404 at `/`, while the application context works. Both containers share `boutique-net`; a Java JDBC client would use `jdbc:mysql://db:3306/boutique` and the matching credentials. The supplied JSP does not make that JDBC connection, and environment variables alone would not add it.

To demonstrate persistence, create a small test table from the MySQL prompt, then run `docker compose down` and `docker compose up -d`; reconnect and confirm the table remains. The `mysql-data` volume stores `/var/lib/mysql`. Host mapping `127.0.0.1:3306:3306` satisfies local access on port 3306; container-to-container connections still use `db:3306`.

When finished, use `docker compose stop` to pause or `docker compose down` to remove this stack. Keep named volumes by omitting `-v`. **Expected result:** the web app opens at its context URL, the containers share their custom network, and the requested port mappings are visible in `docker compose ps`.

### Scenario-based questions and one-line answers

1. **Your Online Boutique Store application runs on Tomcat using port 8080, and MySQL uses port 3306. You want to access Tomcat through the browser and allow the application to communicate with MySQL. How do you configure port mappings and a custom Docker network in Docker Compose?**  
   Publish Tomcat `8080:8080` and MySQL `127.0.0.1:3306:3306`, attach both to one custom network, and use `db:3306` for application database connections.

2. **You are running an Online Boutique Store using Tomcat and MySQL containers. After removing and recreating the MySQL container, you notice that all customer registration and product details are lost. Why does this happen, and how can you configure Docker Compose volumes to preserve database data?**  
   Data was kept only in the removed container’s writable layer; mount a named volume such as `mysql-data:/var/lib/mysql` and retain it when recreating the database.

3. **Your Online Boutique Store is deployed using Tomcat and MySQL containers. When you access the application through http://localhost:8080, you receive a Connection Refused error. How do you identify whether the Tomcat container is running, check its logs, verify port mappings, troubleshoot the database connection, and safely stop and remove the containers?**  
   Check `docker compose ps`, Tomcat/DB logs, `8080:8080` and database settings; use `/OBS-1/`, then safely remove the stack with `docker compose down` without `-v`.

---

## Q2 Jenkins webhooks and Tomcat deployment

### Question

> Q2. Jenkins Pipeline for a Maven Web Project with web hooks[25 Marks]
>
> The Online Boutique Store development team frequently pushes code changes to GitHub. The company wants to trigger the Jenkins Scripted Pipeline automatically whenever developers commit and push changes to the repository. Configure a GitHub webhook and the appropriate Jenkins build trigger to automatically execute the pipeline. The pipeline should perform GitHub clone, Maven clean, build, test, and package operations.
>
> Tasks:
>
> 1. Clone Git Hub repository.( https://github.com/haleema91/LAB-INTERNAL-II-OBS.git)
>
> 2. Configure Tomcat web server.
>
> 3. Deploy the application on the Web Server
>
> 4. Create the Pipeline View using Jenkins and show how code changes triggers jenkins using webhooks.

**One-line explanation:** Use a webhook-triggered Scripted Pipeline to build the boutique WAR and deploy it to local Tomcat.

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
4. The Jenkins account needs write access to `C:\tomcat9\webapps`. The pipeline copies `OBS-1.war` there, and Tomcat auto-deploys it.

#### Step 1 Create the Pipeline job

**Dashboard → New Item** → name `Set6_Build` → **Pipeline → OK**. Under **Pipeline → Definition**, choose **Pipeline script**. Paste the complete Groovy script below and **Save**; storing it in the Jenkins UI avoids adding a Jenkinsfile to the original repository.

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
                    git branch: 'main', url: 'https://github.com/haleema91/LAB-INTERNAL-II-OBS.git'
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
                    bat 'copy /Y target\\OBS-1.war C:\\tomcat9\\webapps\\OBS-1.war'
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
3. On a successful build, open the archived artifact `OBS-1.war`.
4. Open `http://localhost:8085/OBS-1/` and verify the deployed JSP page.

A nonzero `bat` exit throws an error; the `catch` marks failure and rethrows it, so later normal stages are skipped. `finally` still publishes any test report. Empty reports are allowed here because the supplied repository has no Java tests; do not describe this as a passed test suite.

**Troubleshooting:** verify the exact tool names and branch, inspect Console Output for the first Maven error, and confirm the `dir(...)` block contains `pom.xml`. If an older Maven installation selects an incompatible WAR plugin, declare `org.apache.maven.plugins:maven-war-plugin` version `3.4.0` under `<build><plugins>` in a local lab POM; do not patch the original GitHub repository.

#### Configure the GitHub webhook without timed polling

These are instructions for **your own authorized lab repository**. No webhook, fork, commit or push has been created by this answer. If you cannot administer a repository, its owner must perform the GitHub-side steps; a webhook to someone else’s repository cannot be configured from Jenkins alone.

1. For a reproducible push demo, use a personal fork/lab copy of the question’s project; update the Jenkins checkout URL to that copy, keeping the correct branch.
2. In **Set6_Build → Configure → Build Triggers**, tick **GitHub hook trigger for GITScm polling**. Untick scheduled **Poll SCM** and **Build periodically** → **Save**. The trigger’s name includes “polling”, but the poll is initiated by the webhook rather than a cron schedule.
3. Download/install the Windows agent from [ngrok](https://ngrok.com/download), sign in to the ngrok dashboard, copy your agent authtoken, and run `ngrok config add-authtoken YOUR_TOKEN` in PowerShell once. Then run `ngrok http 8080`; use the HTTPS forwarding domain it prints and leave that terminal running. Keep the actual token out of screenshots and source files.
4. In **your lab GitHub repository → Settings → Webhooks → Add webhook**, set **Payload URL:** `https://YOUR_NGROK_DOMAIN/github-webhook/` including the trailing slash; **Content type:** `application/json`; choose **Just the push event**, tick **Active**, and save. If you configure a webhook secret, configure the matching verification secret in your Jenkins GitHub integration.
5. Run one manual Jenkins build to establish the checkout/repository association. In **GitHub → Webhooks → Recent Deliveries**, verify a successful response, and confirm that ngrok shows the incoming request.
6. In your lab copy only, change a harmless line such as README text on the configured branch, commit and push. Do not click **Build Now** for this second run.
7. Confirm a new Jenkins build appears with a GitHub push cause, Console Output shows the new revision, and the appropriate Pipeline/Build Pipeline view updates.
8. Capture the webhook configuration, successful delivery, Jenkins build cause, and Console Output. Do not expose tokens or credentials in screenshots.

**If it does not trigger:** check ngrok is still forwarding to local port 8080, the current HTTPS domain matches the webhook, the endpoint ends `/github-webhook/`, Jenkins is reachable, and the repository/branch matches. A restarted free ngrok tunnel may have a different address. `localhost` cannot be reached directly by GitHub. An already public Jenkins server does not need ngrok. [Jenkins GitHub plugin](https://plugins.jenkins.io/github/)

### Scenario-based questions and one-line answers

1. **Your project manager asks you to remove previously generated build files before building the Online Boutique Store Maven Java Web Application. Which Maven command should you use in the pipeline?**  
   Use `mvn clean` to remove previous Maven build output such as `target/` before rebuilding.

2. **Your Online Boutique Store source code is available in a private GitHub repository. Where should you provide the repository URL in the Jenkins pipeline?**  
   Provide the URL in the Pipeline’s `git` or `checkout` step with a stored `credentialsId`; never place a private GitHub password/token directly in the script.

3. **Your Online Boutique Store application is building successfully, but the WAR file is not generated. Which Maven command should you execute in pipeline code?**  
   Run `mvn package` (or `mvn clean package`) and confirm the POM uses `war` packaging; this repository generates `target/OBS-1.war`.

---

## Q3 Kubernetes scaling and Nagios

### Question

> Q3. Kubernetes – Container Deployment and Auto Scaling and monitoring
>
> [25 MARKS]
>
> During festival seasons, discount sales, and special promotional offers, thousands of customers access the Online Boutique Store simultaneously to browse products, check prices, and place orders. The company wants to deploy multiple application replicas using Kubernetes to handle increased customer traffic and ensure automatic recovery when application containers fail and monitor application availability using Nagios.
>
> Tasks:
>
> Start a Kubernetes cluster using Minikube.
>
> Create a Kubernetes Deployment nginx with two replicas.
>
> Expose the nginx using a Kubernetes Service of type NodePort at port 7070.
>
> Scale the deployment from two to five replicas to handle increased customer traffic.
>
> Verify the deployment, pods, services using appropriate Kubernetes commands.
>
> Install and configure Nagios Core using Docker to monitor the Online Boutique Store application.

**One-line explanation:** Deploy two Nginx replicas, scale to five, expose port 7070 and monitor the HTTP endpoint with Nagios.

### Solution

#### Step 1 Create the local working folder

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set6\q3" | Out-Null
Set-Location "$HOME\se-lab-work\set6\q3"
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

The paper specifically requests an **Nginx Deployment**, so this practical uses the Nginx image rather than pretending that a generic Nginx page is the full business application. Start with 2 replicas and scale to 5.

#### Step 4 Create deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: set6-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: set6-nginx
  template:
    metadata:
      labels:
        app: set6-nginx
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
  name: set6-nginx
spec:
  type: NodePort
  selector:
    app: set6-nginx
  ports:
    - port: 7070
      targetPort: 80
```

Save this file in the current working directory, then apply it:

```powershell
kubectl apply -f .\deployment.yaml
kubectl rollout status deployment/set6-nginx --timeout=180s
kubectl get deployments,pods,services
minikube service set6-nginx --url
```

The Service port is **7070**, while the container listens on **80**. Kubernetes allocates a NodePort, normally 30000–32767; **7070 is not the nodePort**. Open the URL printed by Minikube. On Windows with the Docker driver, keep the Minikube URL command running when it creates a tunnel.

#### Step 5 Access a fixed local port and allow Nagios to check it

In a separate PowerShell window, run this and leave it running:

```powershell
kubectl port-forward --address=0.0.0.0 service/set6-nginx 7070:7070
```

Open **`http://localhost:7070/`**. Binding to `0.0.0.0` makes the forwarding endpoint reachable from the Nagios Docker container through `host.docker.internal`; allow the relevant local firewall traffic if required. For a browser-only test without container access, omit `--address=0.0.0.0`.

#### Step 6 Scale and verify automatic recovery

```powershell
kubectl scale deployment/set6-nginx --replicas=5
kubectl rollout status deployment/set6-nginx --timeout=180s
kubectl get deployment set6-nginx
kubectl get pods -l app=set6-nginx -o wide
$podName = kubectl get pods -l app=set6-nginx -o jsonpath='{.items[0].metadata.name}'
kubectl delete pod $podName
kubectl get pods -l app=set6-nginx -w
```

Wait for **5/5** ready replicas. The ReplicaSet creates a replacement for a deleted Pod; press **Ctrl+C** after observing recovery. If a container merely crashes inside a still-existing Pod, kubelet normally restarts that container according to its restart policy; it does not always create a new Pod. Restart port forwarding if the selected Pod’s replacement interrupts the forwarding session.

#### Step 7 Inspect failures

```powershell
kubectl describe deployment set6-nginx
kubectl describe pods -l app=set6-nginx
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl logs deployment/set6-nginx --tail=60
kubectl get services
kubectl get endpointslices -l kubernetes.io/service-name=set6-nginx
```

For a restarting Pod, use `kubectl logs POD_NAME --previous` and `kubectl describe pod POD_NAME`. Pending Pods commonly lack CPU/memory capacity or have scheduling/volume constraints; an image pull problem normally shows `ImagePullBackOff`, not merely Pending forever. Check events before changing resources.

#### Optional extension Demonstrate real CPU autoscaling

The scale command above is **manual scaling**. To demonstrate an HPA, enable metrics and use the CPU requests already present in the YAML:

```powershell
minikube addons enable metrics-server
kubectl top pods
kubectl autoscale deployment set6-nginx --cpu-percent=60 --min=2 --max=5
kubectl get hpa -w
```

Wait for valid metrics; no load means the HPA may stay at its minimum. It will not necessarily reach 5 replicas without sustained CPU use. Press Ctrl+C after observing it; remove this optional HPA with `kubectl delete hpa set6-nginx` before expecting later manual replica counts to remain fixed. [Kubernetes HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)

**Expected Kubernetes result:** browser access works, the Service routes to ready Pods, scaling reaches 5 replicas when resources permit, and a deleted Pod is replaced. The Nagios section below monitors the forwarded HTTP endpoint. [Minikube access](https://minikube.sigs.k8s.io/docs/handbook/accessing/)

#### Nagios Step 1 Start the monitoring server

In a separate working folder for Nagios, use Windows PowerShell:

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set6\nagios" | Out-Null
Set-Location "$HOME\se-lab-work\set6\nagios"
docker pull jasonrivers/nagios:latest
docker run -d --name set6-nagios --restart unless-stopped -p 8888:80 -e NAGIOS_TIMEZONE=Asia/Kolkata jasonrivers/nagios:latest
docker ps --filter name=set6-nagios
docker logs --tail 50 set6-nagios
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

Create **`lab.cfg`** in the same folder. The application must already be reachable on Windows host port **7070** at **`/`**:

```text
define command {
    command_name check_lab_http
    command_line $USER1$/check_http -H $HOSTADDRESS$ -p 7070 -u / -w 2 -c 5
}
define command {
    command_name check_lab_tcp
    command_line $USER1$/check_tcp -H $HOSTADDRESS$ -p 7070
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
docker cp .\check_lab_metrics.py set6-nagios:/opt/nagios/libexec/check_lab_metrics.py
docker cp .\lab.cfg set6-nagios:/opt/nagios/etc/conf.d/lab.cfg
docker exec set6-nagios /opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
```

Only after the preflight check reports **zero errors**, restart and test:

```powershell
docker restart set6-nagios
docker exec set6-nagios sh -c 'pgrep -a nagios'
docker exec set6-nagios /opt/nagios/libexec/check_http -H host.docker.internal -p 7070 -u / -w 2 -c 5
```

```powershell
docker exec set6-nagios python3 /opt/nagios/libexec/check_lab_metrics.py cpu
docker exec set6-nagios python3 /opt/nagios/libexec/check_lab_metrics.py memory
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

Stop port forwarding with Ctrl+C. Remove only this question’s resources with `kubectl delete service set6-nginx` and `kubectl delete deployment set6-nginx`. Stop monitoring with `docker stop set6-nagios` when finished; do not confuse a stopped monitoring server with an application recovery.

### Scenario-based questions and one-line answers

1. **Your Online Boutique Store is running successfully inside Minikube, but customers cannot access it through a web browser. How will you expose the application externally using a NodePort Service and access it through the Minikube service URL?**  
   Expose Service port 7070 targeting Nginx 80 and open `minikube service set6-nginx --url` or forward `7070:7070`.

2. **The boutique store administrator wants to confirm whether Nagios is running properly and monitoring the application server. Which commands will you execute to verify Nagios service status and configuration?**  
   Check `docker ps`, logs, `docker exec set6-nagios pgrep -a nagios`, and `/opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg`, then inspect Services.

3. **During a festival sale, you scale the Online Boutique Store deployment from two to five replicas. However, some Pods remain in the Pending state. What could be the possible reasons, and how would you troubleshoot them?**  
   Check insufficient CPU/memory, scheduling rules and storage constraints with `kubectl describe pod`, `kubectl describe node` and `kubectl get events`.

---

## Q4 Static Nginx website on AWS EC2

### Question

> Q4. AWS EC2 Deployment – Online Boutique Store [25 Marks]
>
> An online boutique company wants to launch its website on AWS. Create an Ubuntu EC2 instance, connect to it using SSH, install Docker, and deploy the website's index.html file using an Nginx Docker container. Verify the website using the EC2 public IPv4 address.
>
> Task: Deploy the online boutique store on EC2 instance.

**One-line explanation:** Serve an index.html page from an Nginx Docker container on an Ubuntu EC2 public IP.

### Solution

#### Step 1 Launch and connect to Ubuntu EC2

1. **AWS Console → search EC2 → Instances → Launch instances**; choose your lab region and name `set6-app`.
2. **Application and OS Images → Ubuntu Server 24.04 LTS → 64-bit x86**. Choose the small instance allowed by your lab/account, for example `t3.micro`; use `t2.micro` if your paper/lab specifically requires it and it is offered. More memory may be needed for Maven builds. Free Tier eligibility depends on your account and selected resources; do not assume a machine is always free.
3. **Key pair → Create new key pair** → name `set6-key` → RSA → `.pem`; download and keep the key.
4. **Network settings → Edit:** use a public subnet with an Internet Gateway route; enable **Auto-assign public IP**; create a Security Group with **SSH TCP 22 → My IP**; add **HTTP TCP 80** for the static site. For web ports, use **My IP** for your demo or allow public HTTP access when required by the question.
5. Launch; wait for **Running** and all status checks to pass. Select the instance → **Details → Public IPv4 address**; copy it.
6. **Connect → SSH client** shows the SSH command. In Windows PowerShell, switch to the folder containing the downloaded key and run:

```powershell
Set-Location "$HOME\Downloads"
ssh -i .\set6-key.pem ubuntu@PUBLIC_IP
```

Accept the new host key when prompted. If SSH times out, recheck port 22, your current My IP, public IP and subnet route; if it says `Permission denied (publickey)`, verify the key pair and username `ubuntu`. Inbound web access is a separate rule from SSH.

If Windows SSH specifically reports overly open private-key permissions, run these commands in the key folder, then retry SSH:

```powershell
$keyPath = (Resolve-Path '.\set6-key.pem').Path
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

#### Step 3 Create the static website files

This question asks for `index.html` through Nginx, so deploy a static page; copying a JSP file to Nginx would not execute Java/JSP.

```bash
mkdir -p ~/set6-website
cd ~/set6-website
cat > index.html <<'EOF'
<!doctype html>
<html><head><title>Online Boutique Store</title></head>
<body><h1>Online Boutique Store</h1>
<p>Welcome to our shopping website hosted on AWS EC2.</p>
<ul><li>Kurtis</li><li>Sarees</li><li>Dresses</li></ul>
</body></html>
EOF
cat > Dockerfile <<'EOF'
FROM nginx:stable
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
EOF
```

If using your own laptop `index.html`, upload it from a separate Windows PowerShell window with `scp -i .\set6-key.pem .\index.html ubuntu@PUBLIC_IP:~/set6-website/index.html`, then build the image after the file is present.

#### Step 4 Build run and verify

```bash
sudo docker build -t set6-website:1.0 .
sudo docker run -d --name set6-website --restart unless-stopped -p 80:80 set6-website:1.0
sudo docker ps --filter name=set6-website
sudo docker logs --tail 40 set6-website
curl -I http://localhost/
```

Open **`http://PUBLIC_IP/`** from the laptop, using the instance’s actual Public IPv4. If local curl succeeds but the public page does not, check Security Group inbound HTTP 80, the public subnet/Internet Gateway route and the Ubuntu firewall. If port 80 is occupied, inspect `sudo ss -lntp` and stop the conflicting web server. For a changed page, rebuild the image and replace the named container; restarting alone does not rebuild it. **Expected result:** the boutique heading displays through the EC2 public IP. Stop/terminate the dedicated lab instance after saving any needed work.

### Scenario-based questions and one-line answers

1. **The Online Boutique Store administrator has successfully launched an Ubuntu EC2 instance but needs to access it remotely to configure the website. How will you connect to the EC2 instance from your local machine using an SSH key pair? What steps will you follow if the SSH connection fails?**  
   Run `ssh -i KEY.pem ubuntu@PUBLIC_IP`; if it fails, check instance/public IP, Security Group 22/My IP, username/key pair, routing and private-key file permissions.

2. **An online boutique company wants to launch its shopping website on AWS. The administrator needs a Linux server to host the website. How will you launch an Ubuntu EC2 instance, select an appropriate instance type, configure a key pair, and create a Security Group to allow SSH and HTTP traffic?**  
   Use EC2 Launch instances → Ubuntu x86-64 → an allowed small instance → downloaded `.pem` key → public subnet/public IP → Security Group SSH 22/My IP and HTTP 80.

3. **The Online Boutique Store Docker container is running successfully on the AWS EC2 instance, but customers are unable to access the website through a browser. How will you identify the network issue, configure the EC2 Security Group to allow inbound HTTP traffic on port 80, and verify the website using the EC2 public IPv4 address?**  
   Verify `curl localhost`, the container’s `80:80` mapping, inbound HTTP 80, public subnet/Internet Gateway and firewall, then browse `http://PUBLIC_IP/`.

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

