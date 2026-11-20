# SET-9 — Solutions

Original question paper: `SET-9.docx` in MORNING.

**Commands:** Use Windows **CMD** locally. Use `bash` blocks only after SSH into Ubuntu EC2. Save code blocks in the named files.

**Repository note:** The supplied repo has no `RegistrationServlet`; deploying it does not implement registration.

---

### Question 1. Jenkins Scripted Pipeline and triggers

**Question:** 1. Jenkins – Scripted Pipeline & Build Triggers [25 MARKS]

The sports department is implementing a scripted Jenkins pipeline to automate builds for the Sports Management System. The project team wants to ensure the workspace is cleaned before building, Maven dependencies are resolved using the tools block, and code changes automatically trigger builds via GitHub integration.  
Task: Implement and execute a scripted Jenkins pipeline with Maven tool configuration and automated triggers.  
Define the scripted pipeline structure with agent and tools configurations.  
Implement stages for repository cloning (git clone), cleaning (mvn clean), compiling, and packaging.  
Set up build triggers for automated execution.  
Expected Output:  
Successful execution of scripted pipeline stages in Jenkins console log.  
Successful generation of the .war artifact.

**One-line explanation:** Use a Scripted Pipeline to check out, build, test and package the project.

**Solution:**

**Tools:** Manage Jenkins → Plugins: Git, Maven Integration, Pipeline/Stage View as needed. Manage Jenkins → Tools: add JDK named `JDK21` and Maven named `MAVEN-HOME` using their installation folders.

1. New Item → `Set9_Build` → Pipeline → OK → Definition **Pipeline script**. Paste this Scripted Pipeline:

   ```groovy
   node {
       def jdk = tool 'JDK21'
       def mvn = tool 'MAVEN-HOME'
       withEnv(["JAVA_HOME=${jdk}", "PATH+JDK=${jdk}/bin", "PATH+MAVEN=${mvn}/bin"]) {
           try {
               stage('Clean workspace') { deleteDir() }
               dir('app') {
                   stage('Checkout') { git branch: 'main', url: 'https://github.com/sarasrija/SportSystem.git' }
                   stage('Clean') { bat 'call mvn -B clean' }
                   stage('Build') { bat 'call mvn -B compile' }
                   stage('Test') { bat 'call mvn -B test' }
                   stage('Package') { bat 'call mvn -B package' }
                   stage('Archive') { archiveArtifacts 'target/*.war' }
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


2. Save → Build Now → Console Output. Open the job’s **Stage View** and confirm the stages and archived WAR.

3. For builds only when GitHub changes, configure the webhook below rather than Build periodically.

**GitHub webhook settings:**

1. In **Set9_Build → Configure → Build Triggers**, tick **GitHub hook trigger for GITScm polling**; leave Poll SCM/Build periodically off.

2. In CMD run `ngrok http 8080` and keep it open. If needed, first install ngrok and run `ngrok config add-authtoken YOUR_TOKEN`.

3. In **your own lab repo → Settings → Webhooks → Add webhook**, set Payload URL `https://YOUR_NGROK_DOMAIN/github-webhook/`, content type `application/json`, push event, Active → Add webhook. Use a repo you own and change the Jenkins checkout URL to it; leave the paper’s original repo unchanged.

4. Run one initial build, then push a harmless change to your lab repo. Check Webhook Recent Deliveries, the automatically started Jenkins build and Console Output; capture the screenshots requested by the paper.

**Note:** These web repositories have no Java tests; “No tests to run” is not a passed test suite. A failed Maven command stops later stages. 

**Scenario-based questions:**

- **Your manager asks you to clean the workspace before building—which stage or command in the pipeline script takes care of it?** The `Workspace cleanup` stage uses `deleteDir()` to clear the Jenkins workspace; `mvn clean` separately deletes Maven build output such as `target/`.
- **If Maven is not configured globally in Jenkins, which section of the pipeline script will fail first?** The `tool name: "MAVEN-HOME"` lookup fails before the Maven stages; in Declarative syntax the equivalent failure is the Maven `tools` configuration.
- **Instead of running nightly builds, you want this pipeline to trigger only when GitHub changes occur—where will you configure this trigger?** Select GitHub hook trigger for GITScm polling and configure a push webhook in an authorized lab repository; leave Build periodically unticked.
- **If you replace mvn clean with mvn compile, what difference will it make to the project build output?** `mvn compile` compiles source while retaining existing output; `mvn clean` removes previous build output and does not compile it.
- **What is the difference between polling and webhook triggers in Jenkins?** Polling checks Git on a schedule; a webhook sends an event from GitHub so Jenkins checks/reacts when a push occurs.

---

### Question 2. Sports application on AWS EC2

**Question:** 2. AWS – Cloud Deployment using EC2 [25 MARKS]

The university sports council is deploying a Maven web application version of the Sports Management System onto an AWS EC2 Ubuntu instance. The administrator needs to handle dependencies, place the generated .war file into the Tomcat webapps directory, and ensure persistence across reboots.  
Task: Deploy the Maven web application on an AWS EC2 instance using Tomcat and manage cloud configurations.  
Provision an EC2 instance, install Java, Git, and Docker.  
Clone the sports repository and build the project.  
Deploy the artifact to Tomcat and verify public access.  
Expected Output:  
Running EC2 instance hosting the web application accessible via Public IP.

**One-line explanation:** Deploy the application WAR on Ubuntu EC2 and open `/sportssystem/` through the public IP.

**Solution:**

1. **Launch EC2:** AWS Console → EC2 → Launch instances → Ubuntu 24.04 x86-64 → lab-approved small instance (`t2.micro` if required/available). Create/download `set9-key.pem`. Enable public IP in a public subnet; Security Group inbound **SSH 22 → My IP; Custom TCP 8080 → My IP or the required public audience**. Launch and copy Public IPv4. Free Tier depends on your account.

2. **Connect from CMD** in the folder containing your key; replace `PUBLIC_IP`.

   ```cmd
   cd /d "%USERPROFILE%\Downloads"
   ssh -i set9-key.pem ubuntu@PUBLIC_IP
   ```


3. **Inside Ubuntu EC2:** install the tools and clone the repository.

   ```bash
   sudo apt-get update
   sudo apt-get install -y docker.io git openjdk-21-jdk
   sudo systemctl enable --now docker
   cd ~
   git clone --branch main https://github.com/sarasrija/SportSystem.git
   cd SportSystem
   ```


4. **Create Dockerfile:** run `nano Dockerfile`, paste the following, then Ctrl+O → Enter → Ctrl+X.

   ```dockerfile
   FROM maven:3.9-eclipse-temurin-21 AS build
   WORKDIR /app
   COPY pom.xml ./
   COPY src ./src
   RUN mvn -B clean package
   FROM tomcat:9.0-jdk21-temurin
   COPY --from=build /app/target/sportssystem.war /usr/local/tomcat/webapps/sportssystem.war
   EXPOSE 8080
   CMD ["catalina.sh", "run"]
   ```


5. **Build, run and check:** wait for web-server startup.

   ```bash
   sudo docker build -t set9-app:1.0 .
   sudo docker run -d --name set9-app --restart unless-stopped -p 8080:8080 set9-app:1.0
   sudo docker ps
   sudo docker logs --tail 30 set9-app
   curl -I http://localhost:8080/sportssystem/
   ```


6. **Open in your laptop browser:** `http://PUBLIC_IP:8080/sportssystem/`. If local curl works but the browser fails, check inbound port 8080, the current public IP and routing/firewall.

**Scenario-based questions:**

- **You built a Maven project on EC2, and a .war file generated inside target/. Where will you place the .war file for Tomcat execution, and why?** Place it under Tomcat’s `webapps` directory (here `/usr/local/tomcat/webapps/` inside the container), where Tomcat deploys WAR applications.
- **Your Maven web app is deployed to Tomcat on EC2, but accessing it via browser gives a 404 error. What configuration or path issues will you check?** Check the case-sensitive `/sportssystem/` context, WAR placement/extraction, JSP or servlet path mappings, Tomcat compatibility and deployment logs.
- **You have deployed your web app on EC2 successfully, but after restarting your instance, the public IP changed and the app is not opening. What AWS feature helps avoid this issue?** Use an Elastic IP for a stable public IPv4 address; auto-assigned public IPs can change after stop/start, while an ordinary reboot normally retains them.
- **Your Maven application needs external dependencies during build, but EC2 has no internet. What AWS service or change allows the EC2 instance to download dependencies securely?** Give a private-subnet instance outbound access through a NAT Gateway and valid routing/DNS, or use an accessible internal Maven mirror for dependency downloads.
- **What command should you run to ensure a web server service auto-starts on EC2 instance boot?** Use `sudo systemctl enable SERVICE_NAME`; for this deployment enable Docker and run the application container with `--restart unless-stopped`.

---

### Question 3. Kubernetes and Nagios

**Question:** 3. Kubernetes & Nagios – Minikube Pods & Infrastructure Monitoring [25 MARKS]

During annual sports tournaments, unexpected container crashes occur under high load. The administrator deploys the sports application on a local Minikube cluster and configures Nagios inside Docker to monitor server performance, CPU usage, and application availability.  
Task:  
Manage container replicas in Minikube and set up Nagios host/service monitoring.  
Start Minikube using the Docker driver and create a deployment.  
Scale replicas and expose the deployment via a Service.  
Pull and run the Nagios Docker image to monitor server metrics.  
Expected Output:  
Active Minikube pods and operational Nagios dashboard displaying host metrics.

**One-line explanation:** Deploy 2 replicas, expose the application on port 8099, scale to 4 and check it with Nagios.

**Solution:**

1. **Start Minikube:** open Docker Desktop first, then run in CMD.

   ```cmd
   minikube start --driver=docker
   minikube status
   kubectl config use-context minikube
   kubectl get nodes
   ```


2. **Create the application image:** clone the repo in a new working folder and save the Dockerfile below in the clone.

   ```cmd
   mkdir "%USERPROFILE%\se-lab-work\set9\q3"
   cd /d "%USERPROFILE%\se-lab-work\set9\q3"
   git clone --branch main https://github.com/sarasrija/SportSystem.git app
   cd app
   ```


   ```dockerfile
   FROM maven:3.9-eclipse-temurin-21 AS build
   WORKDIR /app
   COPY pom.xml ./
   COPY src ./src
   RUN mvn -B clean package
   FROM tomcat:9.0-jdk21-temurin
   COPY --from=build /app/target/sportssystem.war /usr/local/tomcat/webapps/sportssystem.war
   EXPOSE 8080
   CMD ["catalina.sh", "run"]
   ```


   ```cmd
   minikube image build -t set9-app:1.0 .
   ```


3. **Create 2 replicas** from the image in Minikube.

   ```cmd
   kubectl create deployment set9-app --image=set9-app:1.0 --replicas=2 --image-pull-policy=Never
   kubectl rollout status deployment/set9-app
   ```


4. **Expose the Service on port 8099:** container port is 8080; Kubernetes allocates a valid NodePort.

   ```cmd
   kubectl expose deployment set9-app --type=NodePort --port=8099 --target-port=8080
   kubectl get services
   minikube service set9-app --url
   ```


5. **Open in browser:** use the printed URL with `/sportssystem/` appended, or keep the following command running and open `http://localhost:8099/sportssystem/`. The all-interface binding lets Nagios reach this forwarded port.

   ```cmd
   kubectl port-forward --address=0.0.0.0 service/set9-app 8099:8099
   ```


6. **Scale to 4 replicas and verify** in a second CMD window.

   ```cmd
   kubectl scale deployment/set9-app --replicas=4
   kubectl rollout status deployment/set9-app
   kubectl get deployments,pods,services
   kubectl get pods -o wide
   ```


7. **Check Pod recovery:** copy one Pod name from the list, replace `POD_NAME`, delete it and watch its replacement.

   ```cmd
   kubectl delete pod POD_NAME
   kubectl get pods -l app=set9-app -w
   ```


**Nagios monitoring:** Keep the port-forward CMD window open.

1. **Run Nagios** from CMD; stop an earlier Nagios container if port 8888 is occupied.

   ```cmd
   docker pull jasonrivers/nagios:latest
   docker run -d --name set9-nagios -p 8888:80 jasonrivers/nagios:latest
   docker ps
   ```


2. **Open dashboard:** `http://localhost:8888` → login `nagiosadmin` / `nagios` → Hosts and Services.

3. **CPU, memory, disk and uptime:** save this as `check_lab_metrics.py`. Thresholds are 80% WARNING and 90% CRITICAL.

   ```python
   import sys, time, shutil
   mode = sys.argv[1]
   if mode == "cpu":
       def sample():
           with open("/proc/stat") as f:
               a = list(map(int, f.readline().split()[1:9]))
           return sum(a), a[3] + a[4]
       total1, idle1 = sample()
       time.sleep(1)
       total2, idle2 = sample()
       value = 100 * (1 - (idle2-idle1) / max(1, total2-total1))
   elif mode == "memory":
       with open("/proc/meminfo") as f:
           mem = {line.split(":")[0]: int(line.split()[1]) for line in f}
       value = 100 * (1 - mem["MemAvailable"] / mem["MemTotal"])
   elif mode == "disk":
       disk = shutil.disk_usage("/")
       value = 100 * disk.used / disk.total
   else:
       with open("/proc/uptime") as f:
           seconds = float(f.read().split()[0])
       print(f"OK - uptime {seconds/3600:.1f} hours")
       sys.exit(0)
   state = 2 if value >= 90 else 1 if value >= 80 else 0
   print(f"{['OK', 'WARNING', 'CRITICAL'][state]} - {mode}: {value:.1f}%")
   sys.exit(state)
   ```


4. **Configure monitoring:** save this as `lab.cfg`. The application must be available on host port 8099.

   ```text
   define command {
       command_name check_lab_http
       command_line $USER1$/check_http -H host.docker.internal -p 8099 -u /sportssystem/ -w 2 -c 5
   }
   define service {
       use generic-service
       host_name localhost
       service_description Application HTTP
       check_command check_lab_http
       check_interval 1
       retry_interval 1
   }
   define command {
       command_name check_lab_metric
       command_line /usr/bin/python3 /opt/nagios/libexec/check_lab_metrics.py $ARG1$
   }
   define service {
       use generic-service
       host_name localhost
       service_description Linux cpu
       check_command check_lab_metric!cpu
       check_interval 1
   }
   define service {
       use generic-service
       host_name localhost
       service_description Linux memory
       check_command check_lab_metric!memory
       check_interval 1
   }
   define service {
       use generic-service
       host_name localhost
       service_description Linux disk
       check_command check_lab_metric!disk
       check_interval 1
   }
   define service {
       use generic-service
       host_name localhost
       service_description Linux uptime
       check_command check_lab_metric!uptime
       check_interval 1
   }
   ```


5. **Load configuration:** run the commands below; restart only if validation reports zero errors.

   ```cmd
   docker cp check_lab_metrics.py set9-nagios:/opt/nagios/libexec/check_lab_metrics.py
   docker cp lab.cfg set9-nagios:/opt/nagios/etc/conf.d/lab.cfg
   docker exec set9-nagios /opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
   docker restart set9-nagios
   docker exec set9-nagios pgrep -a nagios
   ```


6. **Verify monitoring:** Services → Application HTTP → wait/force a check. Stop the application or its port-forward command and observe failure; restore it and confirm OK. These are dashboard alerts; email requires separate mail/contact configuration.

**Note:** These metric checks measure the Linux/Docker environment visible inside Nagios, not native Windows or every remote Pod. Use a target-side agent such as NCPA for remote host metrics.

**Note:** This image uses `/opt/nagios`. `host.docker.internal` reaches the Windows host; `localhost` inside the container would check Nagios itself.

**Expected result:** 4 ready replicas and an accessible application URL. If a Pod is Pending, run `kubectl describe pod POD_NAME` and `kubectl get events`; inspect scheduling/resource errors.

**Scenario-based questions:**

- **Your Pod keeps restarting repeatedly (CrashLoopBackOff). What will you do to troubleshoot this issue?** Use `kubectl logs POD --previous`, `kubectl describe pod POD` and events to investigate exceptions, entry points, probes, memory limits and configuration.
- **A Kubernetes pod is stuck in a Pending state. What could be the possible reasons, and how would you troubleshoot it?** Look for resource shortages, scheduling/taint constraints or unbound storage with `kubectl describe pod`, node details and cluster events.
- **You have a Kubernetes Deployment with multiple replicas, and some pods are failing health checks. How would you identify the root cause and fix it?** Inspect failing probe events, application logs, probe port/path and startup timing, then correct the application or probe configuration and observe rollout readiness.
- **You installed Nagios, but the web interface shows "Unable to connect to Nagios process". How do you resolve this?** Validate the Nagios config, correct errors and permissions/status-file issues, confirm the daemon exists, and restart the container or native Nagios service.
- **You added a new host in Nagios, but it's not appearing on the web interface. How do you check and fix this?** Verify the host definition is in an included config directory, run `nagios -v` using this image’s `/opt/nagios` paths, fix errors and reload/restart.

---

### Question 4. Flask, PostgreSQL and Redis using Docker Compose

**Question:** 4. Docker Compose – Multi-Container Orchestration [25 MARKS]

Scenario:  
The sports development team wants to run a multi-container environment containing a Flask web microservice, a PostgreSQL database, and a Redis cache using Docker Compose to speed up local iteration and testing.  
Task: Write and execute a complex docker-compose.yml file managing multiple interdependent services, custom networks, and volumes.  
Define services (web, db, redis) in docker-compose.yml.  
Configure depends _on, custom networks, and persistent storage volumes.  
Test rebuilding and scaling workflows.  
Expected Output:  
Running multi-container stack with persistent data storage.

**One-line explanation:** Run Flask, PostgreSQL and Redis together, then rebuild and scale the web service.

**Solution:**

1. Open **Command Prompt (CMD)** and create the working folder.

   ```cmd
   mkdir "%USERPROFILE%\se-lab-work\set9\q4"
   cd /d "%USERPROFILE%\se-lab-work\set9\q4"
   ```


2. Save **app.py**, **requirements.txt** and **Dockerfile** with these contents, in that order.

   ```python
   import os
   from flask import Flask
   import psycopg
   from redis import Redis

   app = Flask(__name__)

   @app.get("/")
   def home():
       with psycopg.connect(os.environ["DATABASE_URL"]) as conn:
           database_time = conn.execute("SELECT CURRENT_TIMESTAMP").fetchone()[0]
       cache = Redis(host="redis", port=6379, decode_responses=True)
       visits = cache.incr("visits")
       return f"Sports microservice | database time: {database_time} | visits: {visits}"

   if __name__ == "__main__":
       app.run(host="0.0.0.0", port=5000)
   ```


   ```text
   Flask==3.1.2
   psycopg[binary]==3.2.10
   redis==6.4.0
   ```


   ```dockerfile
   FROM python:3.12-slim
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY app.py .
   EXPOSE 5000
   CMD ["python", "app.py"]
   ```


3. Save the following as **docker-compose.yml**.

   ```yaml
   name: set9-compose

   services:
     web:
       build: .
       image: set9-web:1.0
       ports:
         - "8089:5000"
       environment:
         DATABASE_URL: postgresql://labuser:LabPgPass@db:5432/labdb
       depends_on:
         db:
           condition: service_healthy
         redis:
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

     redis:
       image: redis:7-alpine
       command: ["redis-server", "--appendonly", "yes"]
       volumes:
         - redis-data:/data
       healthcheck:
         test: ["CMD", "redis-cli", "ping"]
         interval: 5s
         timeout: 3s
         retries: 20
       networks: [app-net]

   volumes:
     pg-data:
     redis-data:

   networks:
     app-net:
       driver: bridge
   ```


4. Start the services and check their status.

   ```cmd
   docker compose config --quiet
   docker compose up -d --build
   docker compose ps
   docker compose logs --tail 30
   ```


5. Open **http://localhost:8089**. Refresh to see the PostgreSQL timestamp and Redis visit count.

6. Check PostgreSQL and demonstrate persistence; run the final query after the database is healthy again.

   ```cmd
   docker compose exec db psql -U labuser -d labdb -c "CREATE TABLE IF NOT EXISTS demo (id int); INSERT INTO demo VALUES (1);"
   docker compose down
   docker compose up -d
   docker compose ps
   docker compose exec db psql -U labuser -d labdb -c "SELECT * FROM demo;"
   ```


7. For a code change, edit `app.py` locally, save, then rebuild. Use `docker compose down` when finished.

   ```cmd
   docker compose up -d --build web
   ```


8. To run two web instances, replace the web `ports` entry with the range below, then scale and read the assigned ports.

   ```yaml
   ports: ["8091-8095:5000"]
   ```


   ```cmd
   docker compose up -d --scale web=2
   docker compose ps
   ```


**Scenario-based questions:**

- **You have two applications—a Node.js backend and a Python script that handles scheduled jobs. Should you run multiple applications in the same Docker container?** Normally run the backend and scheduled worker in separate containers/services so each has its own lifecycle, logs and scaling.
- **You try to run two containers that both expose port 8080 on the host. What happens, and how can you run both apps?** The second cannot bind the same host IP/port; use distinct published ports or one reverse proxy routing to separate container ports.
- **Your frontend has a new release, but you don’t want to restart the backend or DB containers. How can you update and rebuild only the frontend container?** Use `docker compose up -d --no-deps --build frontend` (replace `frontend` with the actual service name, `web` in this example).
- **Tomcat is listening on port 8080, but you get a "Connection Refused" error in the browser. Why does this happen?** Check Tomcat is running, host-to-container mapping exists, the correct host/IP/port is used and the listener/firewall allows connections; a wrong context path normally gives 404 rather than refusal.
- **You're not sure which container is using port 3000. How do you check this on your host machine?** Use `docker ps --filter publish=3000` to identify a Docker container publishing that host port; use `netstat -ano | findstr :3000` to inspect a Windows listener.

