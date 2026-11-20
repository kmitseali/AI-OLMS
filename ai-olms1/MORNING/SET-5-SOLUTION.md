# SET-5 — Solutions

Original question paper: `SET-5.docx` in MORNING.

**Commands:** Use Windows **CMD** locally. Use `bash` blocks only after SSH into Ubuntu EC2. Save code blocks in the named files.

**Repository note:** The supplied repo has no `donor` handler; `inventory.jsp` is a placeholder.

---

### Question 1. Jenkins Scripted Pipeline

**Question:** 1. Jenkins – Scripted Pipeline & Build Triggers [25 MARKS]

The blood bank organization is setting up a scripted Jenkins pipeline to automate compilation, unit testing, and packaging for the Blood Bank Management System repository. The team needs to handle variable folder names, manage test failures, and execute builds based on specific project requirements.  
Task: Write and configure a scripted Jenkins pipeline with multi-stage build tasks and error handling.  
Configure pipeline script with tool declarations and stage blocks.  
Execute Maven goals (clean, install, test, package) within batch steps.  
Expected Output:  
Console execution logs showing stage progression and artifact packaging

**One-line explanation:** Use a Scripted Pipeline to check out, build, test and package the project.

**Solution:**

**Tools:** Manage Jenkins → Plugins: Git, Maven Integration, Pipeline/Stage View as needed. Manage Jenkins → Tools: add JDK named `JDK21` and Maven named `MAVEN-HOME` using their installation folders.

1. New Item → `Set5_Build` → Pipeline → OK → Definition **Pipeline script**. Paste this Scripted Pipeline:

   ```groovy
   node {
       def jdk = tool 'JDK21'
       def mvn = tool 'MAVEN-HOME'
       withEnv(["JAVA_HOME=${jdk}", "PATH+JDK=${jdk}/bin", "PATH+MAVEN=${mvn}/bin"]) {
           try {
               stage('Clean workspace') { deleteDir() }
               dir('bloodbankapp') {
                   stage('Checkout') { git branch: 'main', url: 'https://github.com/sarasrija/BloodBanksystem.git' }
                   stage('Clean') { bat 'call mvn -B clean' }
                   stage('Install') { bat 'call mvn -B install' }
                   stage('Test') { bat 'call mvn -B test' }
                   stage('Package') { bat 'call mvn -B package' }
                   stage('Archive') { archiveArtifacts 'target/*.war' }
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


2. Save → Build Now → Console Output. Open the job’s **Stage View** and confirm the stages and archived WAR.

**Note:** These web repositories have no Java tests; “No tests to run” is not a passed test suite. A failed Maven command stops later stages. `install` already includes test/package; the later stages repeat them to match the paper.

**Scenario-based questions:**

- **The project folder name is not mavenjava but bloodbankapp—which parts of the script must you edit?** Change the Pipeline’s `dir(...)` value and any hard-coded workspace/report/artifact paths containing `mavenjava` to `bloodbankapp`; the supplied script already uses that directory.
- **If the install stage fails, will the subsequent test and package stages still run in this scripted pipeline?** No: a failed `bat` step throws an error and the normal later Test/Package stages are skipped unless error handling deliberately catches and continues.
- **Your test cases failed, but the pipeline still continued (or failed depending on configuration)—how does Jenkins handle step execution in the test stage?** A nonzero Maven test exit normally fails the `bat` step; `returnStatus`, `catchError` or ignored test failures can allow continuation, so inspect the configured error handling.
- **If you want to print the current build number inside the console logs for auditing, how do you reference it in the pipeline script?** Use `echo "Build number: ${env.BUILD_NUMBER}"` inside the Scripted Pipeline.
- **What is the role of Jenkins in a Continuous Integration pipeline?** Jenkins orchestrates source retrieval, automated builds/tests, artifact collection and result reporting so integration problems are detected early.

---

### Question 2. Blood Bank application on AWS EC2

**Question:** 2. AWS – Cloud Deployment using EC2 [25 MARKS]

The healthcare institution is migrating the Blood Bank Management System to an AWS EC2 instance. Administrators need to deploy static pages or Java web applications, handle local file directory paths, and automate setup routines during instance reboots.  
Task: Deploy and configure web assets and application archives on an AWS EC2 virtual machine.  
Provision Ubuntu EC2 instance, install required tools via SSH.  
Configure web server paths and deployment directories.  
Expected Output:  
Operational web application hosted on AWS EC2 instance.

**One-line explanation:** Deploy the application WAR on Ubuntu EC2 and open `/bloodbank/` through the public IP.

**Solution:**

1. **Launch EC2:** AWS Console → EC2 → Launch instances → Ubuntu 24.04 x86-64 → lab-approved small instance (`t2.micro` if required/available). Create/download `set5-key.pem`. Enable public IP in a public subnet; Security Group inbound **SSH 22 → My IP; Custom TCP 8080 → My IP or the required public audience**. Launch and copy Public IPv4. Free Tier depends on your account.

2. **Connect from CMD** in the folder containing your key; replace `PUBLIC_IP`.

   ```cmd
   cd /d "%USERPROFILE%\Downloads"
   ssh -i set5-key.pem ubuntu@PUBLIC_IP
   ```


3. **Inside Ubuntu EC2:** install the tools and clone the repository.

   ```bash
   sudo apt-get update
   sudo apt-get install -y docker.io git openjdk-21-jdk
   sudo systemctl enable --now docker
   cd ~
   git clone --branch main https://github.com/sarasrija/BloodBanksystem.git
   cd BloodBanksystem
   ```


4. **Create Dockerfile:** run `nano Dockerfile`, paste the following, then Ctrl+O → Enter → Ctrl+X.

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


5. **Build, run and check:** wait for web-server startup.

   ```bash
   sudo docker build -t set5-app:1.0 .
   sudo docker run -d --name set5-app --restart unless-stopped -p 8080:8080 set5-app:1.0
   sudo docker ps
   sudo docker logs --tail 30 set5-app
   curl -I http://localhost:8080/bloodbank/
   ```


6. **Open in your laptop browser:** `http://PUBLIC_IP:8080/bloodbank/`. If local curl works but the browser fails, check inbound port 8080, the current public IP and routing/firewall.

**Scenario-based questions:**

- **You deployed your index.html or static assets to /var/www/html/ directory on EC2, but the web page still isn’t loading. What are two possible issues you would check?** Check that a web server is running and serving `/var/www/html`, and that the Security Group/firewall permits its published HTTP port.
- **You have a simple index.html file on your laptop and you launched an EC2 instance. The instance is running, but when you open the public IP in a browser, the page doesn’t load. What initial steps will you take?** Install/start a web server, upload `index.html` into its document root, allow inbound HTTP 80, and verify local curl before browsing the current public IP.
- **You can access your web application locally on the EC2 instance using curl localhost:8080, but not from your external browser. What AWS setting is likely missing or misconfigured?** The Security Group must allow inbound TCP 8080 from your client; also verify public routing and any OS firewall if that rule is already correct.
- **You want to automate the deployment of your web assets whenever you restart the EC2 instance. Which EC2 feature can be used to run setup commands automatically during instance startup?** Use EC2 user data/cloud-init configured for per-boot execution or an enabled systemd service; ordinary user-data shell scripts normally run only on first launch.
- **Why should SSH access (Port 22) be restricted in security group settings?** SSH controls administration, so restrict port 22 to trusted IPs to reduce exposure to unauthorized login attempts.

---

### Question 3. Minikube and Nagios

**Question:** 3. Kubernetes & Nagios – Minikube Pods & Infrastructure Monitoring [25 MARKS]

During emergency blood requisition surges, system administrators use Minikube to deploy containerized scaling groups and configure Nagios to monitor server uptime, network devices, and application health checks.  
Task: Manage deployments in Minikube and configure Nagios host/service checks.  
Initialize Minikube cluster and manage pod lifecycles.  
Run Nagios container and review host status dashboards.  
Expected Output:  
Scaled Kubernetes deployments and active Nagios monitoring console.

**One-line explanation:** Deploy 2 replicas, expose the application on port 8095, scale to 3 and check it with Nagios.

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
   mkdir "%USERPROFILE%\se-lab-work\set5\q3"
   cd /d "%USERPROFILE%\se-lab-work\set5\q3"
   git clone --branch main https://github.com/sarasrija/BloodBanksystem.git app
   cd app
   ```


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


   ```cmd
   minikube image build -t set5-app:1.0 .
   ```


3. **Create 2 replicas** from the image in Minikube.

   ```cmd
   kubectl create deployment set5-app --image=set5-app:1.0 --replicas=2 --image-pull-policy=Never
   kubectl rollout status deployment/set5-app
   ```


4. **Expose the Service on port 8095:** container port is 8080; Kubernetes allocates a valid NodePort.

   ```cmd
   kubectl expose deployment set5-app --type=NodePort --port=8095 --target-port=8080
   kubectl get services
   minikube service set5-app --url
   ```


5. **Open in browser:** use the printed URL with `/bloodbank/` appended, or keep the following command running and open `http://localhost:8095/bloodbank/`. The all-interface binding lets Nagios reach this forwarded port.

   ```cmd
   kubectl port-forward --address=0.0.0.0 service/set5-app 8095:8095
   ```


6. **Scale to 3 replicas and verify** in a second CMD window.

   ```cmd
   kubectl scale deployment/set5-app --replicas=3
   kubectl rollout status deployment/set5-app
   kubectl get deployments,pods,services
   kubectl get pods -o wide
   ```


7. **Check Pod recovery:** copy one Pod name from the list, replace `POD_NAME`, delete it and watch its replacement.

   ```cmd
   kubectl delete pod POD_NAME
   kubectl get pods -l app=set5-app -w
   ```


**Nagios monitoring:** Keep the port-forward CMD window open.

1. **Run Nagios** from CMD; stop an earlier Nagios container if port 8888 is occupied.

   ```cmd
   docker pull jasonrivers/nagios:latest
   docker run -d --name set5-nagios -p 8888:80 jasonrivers/nagios:latest
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


4. **Configure monitoring:** save this as `lab.cfg`. The application must be available on host port 8095.

   ```text
   define command {
       command_name check_lab_http
       command_line $USER1$/check_http -H host.docker.internal -p 8095 -u /bloodbank/ -w 2 -c 5
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
   docker cp check_lab_metrics.py set5-nagios:/opt/nagios/libexec/check_lab_metrics.py
   docker cp lab.cfg set5-nagios:/opt/nagios/etc/conf.d/lab.cfg
   docker exec set5-nagios /opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
   docker restart set5-nagios
   docker exec set5-nagios pgrep -a nagios
   ```


6. **Verify monitoring:** Services → Application HTTP → wait/force a check. Stop the application or its port-forward command and observe failure; restore it and confirm OK. These are dashboard alerts; email requires separate mail/contact configuration.

**Note:** These metric checks measure the Linux/Docker environment visible inside Nagios, not native Windows or every remote Pod. Use a target-side agent such as NCPA for remote host metrics.

**Note:** This image uses `/opt/nagios`. `host.docker.internal` reaches the Windows host; `localhost` inside the container would check Nagios itself.

**Expected result:** 3 ready replicas and an accessible application URL. If a Pod is Pending, run `kubectl describe pod POD_NAME` and `kubectl get events`; inspect scheduling/resource errors.

**Scenario-based questions:**

- **How would you debug a failed deployment in Kubernetes using kubectl diagnostic commands?** Use `kubectl describe deployment`, `kubectl describe pod`, `kubectl logs` and sorted cluster events to find image, scheduling, configuration or application failures.
- **How do you roll back a faulty deployment in Kubernetes if an update introduces bugs?** Use `kubectl rollout undo deployment/NAME`, then `kubectl rollout status deployment/NAME`; a previous recorded revision must exist.
- **How do you debug a running Pod when application-level errors occur inside the container?** Read `kubectl logs POD` and `--previous` for prior crashes, inspect `kubectl describe pod POD`, and use `kubectl exec -it POD -- sh` when a shell is available.
- **How can you check whether the Nagios monitoring daemon/service is running properly on your system?** For this Docker install use `docker exec set5-nagios pgrep -a nagios`, logs and configuration validation; for a native systemd installation use `systemctl status nagios`.
- **What are the main advantages of using Nagios for IT infrastructure monitoring?** Nagios centralizes host/service checks, configurable thresholds, alerts and recovery tracking to detect infrastructure failures promptly.

---

### Question 4. Docker Compose with PostgreSQL

**Question:** 4. Docker Compose – Multi-Container Orchestration [25 MARKS

The blood bank engineering team is standardizing local deployments using Docker Compose to orchestrate a web frontend container and a PostgreSQL backend container together with persistent storage volumes and custom networks.  
Task: Create and run a multi-container stack using a docker-compose.yml configuration file.  
Define services, networks, and volumes in YAML.  
Execute container start, stop, and clean-up commands.  
Expected Output:  
Running multi-container stack with persistent data storage and inter-service communication.

**One-line explanation:** Run the Blood Bank web application and PostgreSQL with a shared network and persistent volume.

**Solution:**

1. Open **Command Prompt (CMD)** and create the working folder.

   ```cmd
   mkdir "%USERPROFILE%\se-lab-work\set5\q4"
   cd /d "%USERPROFILE%\se-lab-work\set5\q4"
   ```


2. Clone the source into `app`, then save the following as **Dockerfile** beside `docker-compose.yml`.

   ```cmd
   git clone --branch main https://github.com/sarasrija/BloodBanksystem.git app
   ```


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


3. Save the following as **docker-compose.yml**.

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


4. Start the services and check their status.

   ```cmd
   docker compose config --quiet
   docker compose up -d --build
   docker compose ps
   docker compose logs --tail 30
   ```


5. Open **http://localhost:8085/bloodbank/**. Confirm the application page loads.

6. Check PostgreSQL and demonstrate persistence; run the final query after the database is healthy again.

   ```cmd
   docker compose exec db psql -U labuser -d labdb -c "CREATE TABLE IF NOT EXISTS demo (id int); INSERT INTO demo VALUES (1);"
   docker compose down
   docker compose up -d
   docker compose ps
   docker compose exec db psql -U labuser -d labdb -c "SELECT * FROM demo;"
   ```


7. For a code change, edit `app/src/main/webapp/index.jsp` locally, save, then rebuild. Use `docker compose down` when finished.

   ```cmd
   docker compose up -d --build web
   ```


**Note:** The JSP repository has no PostgreSQL connection code; the SQL check verifies the backend environment separately.

**Scenario-based questions:**

- **How do you stop and clean up your running container stack and remove associated resources without losing volume data?** Run `docker compose down` without `-v`; it removes the stack’s containers/network while retaining named volume data.
- **You updated some code and want to restart your frontend, backend, and DB containers quickly. What is the easiest way to manage this lifecycle?** Use `docker compose restart` for a quick restart, or `docker compose up -d --build` when changed code is baked into the application images.
- **How do you share your application configuration and code as an image bundle with a teammate using Docker images?** Export images with `docker save -o bundle.tar IMAGE...`, share that archive plus the Compose/config files, and import with `docker load -i bundle.tar`; volumes require a separate backup.
- **What happens when you try to run two containers that expose the exact same port on the host machine?** The second container cannot bind the same host IP/port, so choose different host ports or put both behind a reverse proxy.
- **Why are networks specified in Docker Compose instead of relying solely on default bridge networking for interdependent services?** Compose-managed bridge networks provide service-name DNS and project isolation; its default project network already does this, while explicit networks let you control service grouping.

