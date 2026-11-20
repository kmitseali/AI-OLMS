# SET-7 — Solutions

Original question paper: `SET-7.docx` in MORNING.

**Commands:** Use Windows **CMD** locally. Use `bash` blocks only after SSH into Ubuntu EC2. Save code blocks in the named files.

**Repository note:** The supplied repo has no `tenant` handler; that link cannot work without its missing code.

---

### Question 1. Jenkins CI, Tomcat and webhook

**Question:** 1. Jenkins – Web Application Build and Continuous Integration [25 MARKS]

A development team working on the web-based Apartment Management System repository wants to automate compilation, unit testing, and artifact generation using Jenkins. To avoid manual polling, the team needs to configure GitHub webhooks using ngrok so that every code push automatically triggers the build pipeline and displays results in the Jenkins Build History / Pipeline View.  
Task: Implement an automated Continuous Integration workflow using Jenkins with GitHub Webhook integration.  
1. Clone Git Hub repository and Configure Tomcat web server.  
3. Deploy the application on the Web Server  
4. Create the Pipeline View using Jenkins  
Expected Output:  
Jenkins files as necessary  
Jenkins pipeline view.

**One-line explanation:** Use a Scripted Pipeline to check out, build, test and package the project and deploy its WAR to Tomcat.

**Solution:**

**Tools:** Manage Jenkins → Plugins: Git, Maven Integration, Pipeline/Stage View as needed. Manage Jenkins → Tools: add JDK named `JDK21` and Maven named `MAVEN-HOME` using their installation folders.

1. **Clone:** the Checkout stage below uses `https://github.com/sarasrija/ApartmentSystem.git` on branch `main`.

2. **Configure Tomcat:** Extract Tomcat 9 so `C:\tomcat9\bin` and `webapps` exist. Edit `conf\server.xml`: change the HTTP Connector from 8080 to **8085**. Start it with `C:\tomcat9\bin\startup.bat`; allow the Jenkins account to write to `C:\tomcat9\webapps`.

3. **Build and deploy:** New Item → `Set7_Build` → Pipeline → Pipeline script → paste this script → Save. Install Pipeline: Stage View if missing.

   ```groovy
   node {
       def jdk = tool 'JDK21'
       def mvn = tool 'MAVEN-HOME'
       withEnv(["JAVA_HOME=${jdk}", "PATH+JDK=${jdk}/bin", "PATH+MAVEN=${mvn}/bin"]) {
           try {
               stage('Clean workspace') { deleteDir() }
               dir('app') {
                   stage('Checkout') { git branch: 'main', url: 'https://github.com/sarasrija/ApartmentSystem.git' }
                   stage('Clean') { bat 'call mvn -B clean' }
                   stage('Build') { bat 'call mvn -B compile' }
                   stage('Test') { bat 'call mvn -B test' }
                   stage('Package') { bat 'call mvn -B package' }
                   stage('Archive') { archiveArtifacts 'target/*.war' }
                   stage('Deploy') { bat 'copy /Y target\\apartmentsystem.war C:\\tomcat9\\webapps\\apartmentsystem.war' }
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


4. **Pipeline View and trigger:** Build Now → Console Output; check Stage View and `http://localhost:8085/apartmentsystem/`. Configure the webhook below.

**GitHub webhook settings:**

1. In **Set7_Build → Configure → Build Triggers**, tick **GitHub hook trigger for GITScm polling**; leave Poll SCM/Build periodically off.

2. In CMD run `ngrok http 8080` and keep it open. If needed, first install ngrok and run `ngrok config add-authtoken YOUR_TOKEN`.

3. In **your own lab repo → Settings → Webhooks → Add webhook**, set Payload URL `https://YOUR_NGROK_DOMAIN/github-webhook/`, content type `application/json`, push event, Active → Add webhook. Use a repo you own and change the Jenkins checkout URL to it; leave the paper’s original repo unchanged.

4. Run one initial build, then push a harmless change to your lab repo. Check Webhook Recent Deliveries, the automatically started Jenkins build and Console Output; capture the screenshots requested by the paper.

**Note:** These web repositories have no Java tests; “No tests to run” is not a passed test suite. A failed Maven command stops later stages. 

**Scenario-based questions:**

- **After setting up the GitHub webhook , you push code changes, but Jenkins does not trigger automatically. What simple check would you perform on the ngrok terminal session to ensure it is still running and forwarding requests?** Confirm ngrok still displays an active HTTPS forwarding URL to `http://localhost:8080` and receives webhook requests; update your lab webhook if its domain changed.
- **How does Jenkins display compilation errors in the console output when a syntax mistake is made in a Java file?** Maven prints `[ERROR]` compiler messages with the source file and line, exits nonzero, and Jenkins marks the build FAILED in Console Output.
- **If Maven successfully compiles files , but a unit test fails during , will Jenkins still go ahead and create the .war file?** Normally no: Maven stops at the failing test phase before packaging; ignoring test failures or retaining an older WAR can obscure this, so clean and inspect the result.
- **Why is ngrok necessary when connecting a local Jenkins server to GitHub webhooks?** ngrok supplies a public HTTPS route to local Jenkins because GitHub cannot send webhooks to your laptop’s private localhost.
- **What is the difference between Continuous Integration and Continuous Deployment?** Continuous Integration automatically builds/tests changes; Continuous Deployment also automatically releases validated changes to the target environment.

---

### Question 2. Apartment application on AWS EC2

**Question:** 2. AWS – Cloud Deployment using EC2 [25 MARKS]

The apartment association wants to migrate the Apartment Management System from a local server to an AWS EC2 instance so residents and administrators can access the portal remotely over the internet. The deployment requires provisioning an Ubuntu virtual machine, installing Docker and Git, writing a Dockerfile to package the application with Tomcat, and securing network access.  
Task: Deploy the Apartment Management application on AWS using an EC2 instance and Docker.  
Expected Output:  
Launch an Ubuntu EC2 instance (t2.micro, free tier) and connect via SSH using the .pem key.  
Update system packages, install Docker and Git .  
Clone the GitHub repository onto the instance and Write a custom Docker file.  
Build the Docker image, run it as a detached container mapping host port to container port 8090, and verify access via the EC2 public IP.

**One-line explanation:** Deploy the application WAR on Ubuntu EC2 and open `/apartmentsystem/` through the public IP.

**Solution:**

1. **Launch EC2:** AWS Console → EC2 → Launch instances → Ubuntu 24.04 x86-64 → lab-approved small instance (`t2.micro` if required/available). Create/download `set7-key.pem`. Enable public IP in a public subnet; Security Group inbound **SSH 22 → My IP; Custom TCP 8090 → My IP or the required public audience**. Launch and copy Public IPv4. Free Tier depends on your account.

2. **Connect from CMD** in the folder containing your key; replace `PUBLIC_IP`.

   ```cmd
   cd /d "%USERPROFILE%\Downloads"
   ssh -i set7-key.pem ubuntu@PUBLIC_IP
   ```


3. **Inside Ubuntu EC2:** install the tools and clone the repository.

   ```bash
   sudo apt-get update
   sudo apt-get install -y docker.io git openjdk-21-jdk
   sudo systemctl enable --now docker
   cd ~
   git clone --branch main https://github.com/sarasrija/ApartmentSystem.git
   cd ApartmentSystem
   ```


4. **Create Dockerfile:** run `nano Dockerfile`, paste the following, then Ctrl+O → Enter → Ctrl+X.

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


5. **Build, run and check:** wait for web-server startup.

   ```bash
   sudo docker build -t set7-app:1.0 .
   sudo docker run -d --name set7-app --restart unless-stopped -p 8090:8090 set7-app:1.0
   sudo docker ps
   sudo docker logs --tail 30 set7-app
   curl -I http://localhost:8090/apartmentsystem/
   ```


6. **Open in your laptop browser:** `http://PUBLIC_IP:8090/apartmentsystem/`. If local curl works but the browser fails, check inbound port 8090, the current public IP and routing/firewall.

**Scenario-based questions:**

- **Your container is running on port 8090 inside EC2, but you try to open port 8080 in your browser URL. What type of connection error do you get, and which AWS setting must you update in the Security Group?** Port 8080 has no matching listener so it is refused or times out; browse the published 8090 URL and allow inbound TCP 8090 in the Security Group.
- **Why is leaving SSH access (Port 22) open to everyone (0.0.0.0/0) considered a security risk in AWS Security Groups?** It exposes the administrative login endpoint to the whole internet, increasing the chance of unauthorized attempts; use trusted source IPs and key authentication.
- **What are the basic Linux shell commands used to refresh packages and install Docker on a new Ubuntu EC2 instance?** Run `sudo apt-get update` followed by `sudo apt-get install -y docker.io git` and `sudo systemctl enable --now docker`.
- **If the Tomcat server inside your running container crashes unexpectedly, how can you view the container logs to find out what went wrong?** Run `sudo docker logs --tail 100 set7-app` or `sudo docker logs -f set7-app` to inspect Tomcat startup and failure messages.
- **How can application availability be improved?** Use health checks, process/container restart policies, multiple instances across Availability Zones, a load balancer, Auto Scaling and monitoring.

---

### Question 3. Kubernetes scaling and Nagios

**Question:** 3. Kubernetes – Container Deployment and Auto Scaling [25 MARKS]

Scenario: Managing Heavy Traffic During Monthly Maintenance Deadlines  
On the final day of every month, thousands of residents simultaneously log in to the Apartment Management Portal to pay monthly maintenance bills and book facilities. To handle this surge and ensure high availability, the administration wants to deploy multiple application replicas using Kubernetes (Minikube) and ensure automatic recovery when containers fail.  
Task: Deploy, scale, and manage the application using Kubernetes.  
Expected Output:  
Start a local Kubernetes cluster using Minikube .  
Create a Deployment for the apartment application with multiple 4 replicas  
Expose the application using a Kubernetes Service.  
Scale the deployment to handle higher loads.  
Pull and run the Nagios Docker image and run the container.  
Monitor CPU, memory, disk and server uptime.

**One-line explanation:** Deploy 4 replicas, expose the application on port 8097, scale to 5 and check it with Nagios.

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
   mkdir "%USERPROFILE%\se-lab-work\set7\q3"
   cd /d "%USERPROFILE%\se-lab-work\set7\q3"
   git clone --branch main https://github.com/sarasrija/ApartmentSystem.git app
   cd app
   ```


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


   ```cmd
   minikube image build -t set7-app:1.0 .
   ```


3. **Create 4 replicas** from the image in Minikube.

   ```cmd
   kubectl create deployment set7-app --image=set7-app:1.0 --replicas=4 --image-pull-policy=Never
   kubectl rollout status deployment/set7-app
   ```


4. **Expose the Service on port 8097:** container port is 8090; Kubernetes allocates a valid NodePort.

   ```cmd
   kubectl expose deployment set7-app --type=NodePort --port=8097 --target-port=8090
   kubectl get services
   minikube service set7-app --url
   ```


5. **Open in browser:** use the printed URL with `/apartmentsystem/` appended, or keep the following command running and open `http://localhost:8097/apartmentsystem/`. The all-interface binding lets Nagios reach this forwarded port.

   ```cmd
   kubectl port-forward --address=0.0.0.0 service/set7-app 8097:8097
   ```


6. **Scale to 5 replicas and verify** in a second CMD window.

   ```cmd
   kubectl scale deployment/set7-app --replicas=5
   kubectl rollout status deployment/set7-app
   kubectl get deployments,pods,services
   kubectl get pods -o wide
   ```


7. **Check Pod recovery:** copy one Pod name from the list, replace `POD_NAME`, delete it and watch its replacement.

   ```cmd
   kubectl delete pod POD_NAME
   kubectl get pods -l app=set7-app -w
   ```


**Nagios monitoring:** Keep the port-forward CMD window open.

1. **Run Nagios** from CMD; stop an earlier Nagios container if port 8888 is occupied.

   ```cmd
   docker pull jasonrivers/nagios:latest
   docker run -d --name set7-nagios -p 8888:80 jasonrivers/nagios:latest
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


4. **Configure monitoring:** save this as `lab.cfg`. The application must be available on host port 8097.

   ```text
   define command {
       command_name check_lab_http
       command_line $USER1$/check_http -H host.docker.internal -p 8097 -u /apartmentsystem/ -w 2 -c 5
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
   docker cp check_lab_metrics.py set7-nagios:/opt/nagios/libexec/check_lab_metrics.py
   docker cp lab.cfg set7-nagios:/opt/nagios/etc/conf.d/lab.cfg
   docker exec set7-nagios /opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
   docker restart set7-nagios
   docker exec set7-nagios pgrep -a nagios
   ```


6. **Verify monitoring:** Services → Application HTTP → wait/force a check. Stop the application or its port-forward command and observe failure; restore it and confirm OK. These are dashboard alerts; email requires separate mail/contact configuration.

**Note:** These metric checks measure the Linux/Docker environment visible inside Nagios, not native Windows or every remote Pod. Use a target-side agent such as NCPA for remote host metrics.

**Note:** This image uses `/opt/nagios`. `host.docker.internal` reaches the Windows host; `localhost` inside the container would check Nagios itself.

**Expected result:** 5 ready replicas and an accessible application URL. If a Pod is Pending, run `kubectl describe pod POD_NAME` and `kubectl get events`; inspect scheduling/resource errors.

**Scenario-based questions:**

- **If one of your application pods stops working suddenly due to a crash, how does Kubernetes handle it to maintain the required number of replicas?** Kubelet restarts a crashed container; when a Pod is lost/deleted, the Deployment’s ReplicaSet creates a replacement to restore the desired replicas.
- **What is the basic difference between a Pod IP address and a Service IP address when routing traffic?** A Pod IP identifies one ephemeral Pod, while a Service provides stable addressing and routes to matching ready backends.
- **Which kubectl command is used to check the list of currently running pods and their statuses?** Use `kubectl get pods` or `kubectl get pods -o wide`.
- **If you cannot open the NodePort page directly in your browser, what alternative command (kubectl port-forward) can you use to test the app locally?** Use `kubectl port-forward service/set7-app 8097:8097`, then open `http://localhost:8097/apartmentsystem/`.
- **How can Nagios identify application failure when the server is running?** Use an application-level HTTP check for the correct port/path so a reachable server can still report a failed application service.

---

### Question 4. Docker Compose with PostgreSQL

**Question:** 4. Docker Compose – Multi-Container Application [25 MARKS]

Scenario: Multi-Container Orchestration for Apartment Community Portal  
The development team needs to run the Apartment Management System web frontend alongside a backend PostgreSQL database (postgres:15) as an integrated stack. To streamline local development, ensure consistent environments, and handle data persistence across restarts, the team must define and execute the setup using Docker Compose.  
Task: Define and run interdependent multi-container services using docker-compose.yml.  
Expected Output:  
Create a working directory (compose-lab) containing a docker-compose.yml file (version 3.9).  
Define a web service (Nginx/Tomcat) and a db service (postgres:15) with appropriate environment variables.  
Configure custom container networking (networks) so services can communicate securely.  
Configure persistent storage volumes (volumes) so database records persist after container removal .  
Run the stack in detached mode , verify container status, and test rebuilding after code changes.

**One-line explanation:** Run the Apartment web application and PostgreSQL using Compose version 3.9.

**Solution:**

1. Open **Command Prompt (CMD)** and create the working folder.

   ```cmd
   mkdir "%USERPROFILE%\se-lab-work\set7\q4"
   cd /d "%USERPROFILE%\se-lab-work\set7\q4"
   mkdir compose-lab
   cd compose-lab
   ```


2. Clone the source into `app`, then save the following as **Dockerfile** beside `docker-compose.yml`.

   ```cmd
   git clone --branch main https://github.com/sarasrija/ApartmentSystem.git app
   ```


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


3. Save the following as **docker-compose.yml**.

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


4. Start the services and check their status.

   ```cmd
   docker compose config --quiet
   docker compose up -d --build
   docker compose ps
   docker compose logs --tail 30
   ```


5. Open **http://localhost:8087/apartmentsystem/**. Confirm the application page loads.

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

- **What is the main use of the depends_on setting in a docker-compose.yml file?** `depends_on` controls startup order; combine `condition: service_healthy` with a database healthcheck when the frontend must wait for database readiness.
- **Why do we configure persistent volumes for databases in Docker Compose, and what happens to saved records if volumes are left out when running docker compose down?** Named volumes preserve database files when containers are removed; without persistent storage, removal of the database container loses data kept only in its writable layer.
- **When you update your source code and run docker compose up -d, why don't the changes show up right away, and what flag should you add to rebuild everything?** Changed files baked into an image need rebuilding; use `docker compose up -d --build`.
- **How do custom networks help containers find and talk to each other using names instead of IP addresses?** Containers on the same user-defined Compose network resolve service names such as `db` through Docker DNS instead of relying on changing IPs.
- **How can database information be recovered after container deletion?** Recreate MySQL/PostgreSQL with the original retained data volume mounted at its database data directory, or restore a backup if the volume was deleted.

