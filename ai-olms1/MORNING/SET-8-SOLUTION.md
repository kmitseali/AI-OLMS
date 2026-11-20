# SET-8 — Solutions

Original question paper: `SET-8.docx` in MORNING.

**Commands:** Use Windows **CMD** locally. Use `bash` blocks only after SSH into Ubuntu EC2. Save code blocks in the named files.

---

### Question 1. Vehicle Rental Java application on AWS EC2

**Question:** Q1. AWS EC2 Deployment – Online Vehicle Rental Maven Java Application [25 Marks]

An online vehicle rental company has developed a Maven-based Java standalone application to manage vehicle rental details. The application allows administrators to add vehicle details, display available vehicles, calculate rental charges, and manage customer bookings.  
The company wants to deploy this Java standalone application on an AWS EC2 Ubuntu instance. The administrator must install git,Docker. Create a Docker image and execute the Java application inside a Docker container.  
Tasks:  
Launch an Ubuntu EC2 instance on AWS.  
Configure an appropriate instance type, key pair and Security Group.  
Clone the Online Vehicle Rental Maven Java standalone project from GitHub.( https://github.com/haleema91/LAB-INTERNAL-II-OVR1.git)  
Create a Dockerfile.  
Build the Docker image for the Maven Java standalone application.  
Run the Docker container and verify the application execution through container logs.

**One-line explanation:** Deploy the Vehicle Rental console application in Docker on Ubuntu EC2.

**Solution:**

1. **Launch EC2:** AWS Console → EC2 → Launch instances → Ubuntu 24.04 x86-64 → lab-approved small instance (`t2.micro` if required/available). Create/download `set8-key.pem`. Enable public IP in a public subnet; Security Group inbound **SSH 22 → My IP**. Launch and copy Public IPv4. Free Tier depends on your account.

2. **Connect from CMD** in the folder containing your key; replace `PUBLIC_IP`.

   ```cmd
   cd /d "%USERPROFILE%\Downloads"
   ssh -i set8-key.pem ubuntu@PUBLIC_IP
   ```


3. **Inside Ubuntu EC2:** install the tools and clone the repository.

   ```bash
   sudo apt-get update
   sudo apt-get install -y docker.io git openjdk-21-jdk
   sudo systemctl enable --now docker
   cd ~
   git clone --branch main https://github.com/haleema91/LAB-INTERNAL-II-OVR1.git
   cd LAB-INTERNAL-II-OVR1
   ```


4. **Create Dockerfile:** run `nano Dockerfile`, paste the following, then Ctrl+O → Enter → Ctrl+X. The `jar` command adds the missing executable main-class entry.

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


5. **Build and execute:** enter `Asha`, vehicle `1`, and `2` rental days when prompted.

   ```bash
   sudo docker build -t set8-rental:1.0 .
   sudo docker run -it --name set8-rental set8-rental:1.0
   ```


6. **Verify logs/status:** expect Toyota Innova, total 5000.0, CONFIRMED and Exited (0). A console program normally finishes; no browser port is needed.

   ```bash
   sudo docker ps -a --filter name=set8-rental
   sudo docker logs set8-rental
   sudo docker inspect set8-rental --format '{{.State.ExitCode}}'
   ```


**Scenario-based questions:**

- **The Online Vehicle Rental administrator has successfully launched an Ubuntu EC2 instance but needs to access it remotely. How will you connect to the EC2 instance from your local machine using an SSH key pair? What steps will you follow if the SSH connection fails?** Connect with `ssh -i KEY.pem ubuntu@PUBLIC_IP`; troubleshoot key/username, key-file permissions, running status, current public IP, SSH/My IP rule and public routing.
- **The development team has uploaded the Online Vehicle Rental Maven Java standalone project to GitHub. The administrator needs to clone the repository and generate the executable JAR file. Which Maven commands will you execute to clean previous build files, compile the Java source code, execute test cases, and package the application into a JAR file?** After cloning the repository, run `mvn clean compile`, `mvn test`, and `mvn package` (or simply `mvn clean package`); add a Main-Class manifest or use `java -cp` for this original JAR.
- **The Docker image has been created successfully, but the administrator wants to verify whether the Online Vehicle Rental Java application is executing properly inside the container. Which Docker commands will you use to run the container, check its status, inspect application logs, and stop the container?** Use `docker run -it --name rental IMAGE`, `docker ps -a`, `docker logs rental` and `docker stop rental`; normal completion of this console task produces Exited (0).

---

### Question 2. Kubernetes Nginx scaling and Nagios

**Question:** Q2. Kubernetes – Container Deployment, Auto Scaling and Monitoring – Online Vehicle rental [25 Marks]

An automobile service management company has developed a Maven-based Java application to manage vehicle servicing, customer information, service appointments, and maintenance records.  
During festive seasons and promotional service campaigns, a large number of customers access the application simultaneously to schedule vehicle servicing, check service status and view maintenance details.  
The company wants to deploy its Java application using Kubernetes to handle increased customer traffic, maintain application availability through automatic Pod recovery and monitor application availability using Nagios.  
Tasks:  
Start a Kubernetes cluster using Minikube.  
Create a Kubernetes Deployment nginx with three replicas.  
Expose the deployment using a Kubernetes Service of type NodePort at port 6060.  
Scale the deployment from three to five replicas to handle increased customer traffic.  
Verify deployments, Pods and Services using appropriate Kubernetes commands.  
Install and configure Nagios Core using Docker to monitor the Online Vehicle Service Management application.

**One-line explanation:** Deploy 3 replicas, expose the application on port 6060, scale to 5 and check it with Nagios.

**Solution:**

1. **Start Minikube:** open Docker Desktop first, then run in CMD.

   ```cmd
   minikube start --driver=docker
   minikube status
   kubectl config use-context minikube
   kubectl get nodes
   ```


2. **Create the Nginx Deployment with 3 replicas.**

   ```cmd
   kubectl create deployment set8-nginx --image=nginx:stable --replicas=3
   kubectl rollout status deployment/set8-nginx
   ```


3. **Expose the Service on port 6060:** container port is 80; Kubernetes allocates a valid NodePort.

   ```cmd
   kubectl expose deployment set8-nginx --type=NodePort --port=6060 --target-port=80
   kubectl get services
   minikube service set8-nginx --url
   ```


4. **Open in browser:** use the printed URL, or keep the following command running and open `http://localhost:6060/`. The all-interface binding lets Nagios reach this forwarded port.

   ```cmd
   kubectl port-forward --address=0.0.0.0 service/set8-nginx 6060:6060
   ```


5. **Scale to 5 replicas and verify** in a second CMD window.

   ```cmd
   kubectl scale deployment/set8-nginx --replicas=5
   kubectl rollout status deployment/set8-nginx
   kubectl get deployments,pods,services
   kubectl get pods -o wide
   ```


**Nagios monitoring:** Keep the port-forward CMD window open.

1. **Run Nagios** from CMD; stop an earlier Nagios container if port 8888 is occupied.

   ```cmd
   docker pull jasonrivers/nagios:latest
   docker run -d --name set8-nagios -p 8888:80 jasonrivers/nagios:latest
   docker ps
   ```


2. **Open dashboard:** `http://localhost:8888` → login `nagiosadmin` / `nagios` → Hosts and Services.

3. **Configure monitoring:** save this as `lab.cfg`. The application must be available on host port 6060.

   ```text
   define command {
       command_name check_lab_http
       command_line $USER1$/check_http -H host.docker.internal -p 6060 -u / -w 2 -c 5
   }
   define service {
       use generic-service
       host_name localhost
       service_description Application HTTP
       check_command check_lab_http
       check_interval 1
       retry_interval 1
   }
   ```


4. **Load configuration:** run the commands below; restart only if validation reports zero errors.

   ```cmd
   docker cp lab.cfg set8-nagios:/opt/nagios/etc/conf.d/lab.cfg
   docker exec set8-nagios /opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
   docker restart set8-nagios
   docker exec set8-nagios pgrep -a nagios
   ```


5. **Verify monitoring:** Services → Application HTTP → wait/force a check. Stop the application or its port-forward command and observe failure; restore it and confirm OK. These are dashboard alerts; email requires separate mail/contact configuration.

**Note:** This image uses `/opt/nagios`. `host.docker.internal` reaches the Windows host; `localhost` inside the container would check Nagios itself.

**Expected result:** 5 ready replicas and an accessible application URL. If a Pod is Pending, run `kubectl describe pod POD_NAME` and `kubectl get events`; inspect scheduling/resource errors.

**Scenario-based questions:**

- **The Online Vehicle Service Management application has been deployed successfully inside Minikube, but customers are unable to access the application through a web browser. How will you expose the application externally using a NodePort Service and access it through the Minikube service URL? What configurations are required in the Docker image and Kubernetes Service?** Configure Service port 6060 with `targetPort: 80` for Nginx, keep labels/selectors matching, and use `minikube service set8-nginx --url` or port forwarding; a console JAR itself is not an HTTP server.
- **During a special vehicle servicing campaign, the administrator scales the Online Vehicle Service Management deployment from two to four replicas. However, some Pods remain in the Pending state. What could be the possible reasons, and how would you troubleshoot the issue using Kubernetes commands?** Inspect resources, node scheduling/taint rules and unbound storage with `kubectl describe pod`, `kubectl describe node` and sorted `kubectl get events`.
- **The company administrator wants to monitor the availability of the Online Vehicle Service Management application deployed in Kubernetes. How will you install and configure Nagios Core using Docker? Which commands will you execute to verify Nagios service status and application monitoring?** Run the Nagios image on 8888, load a valid HTTP host/service config under `/opt/nagios/etc/conf.d`, validate it, restart and check container/daemon status plus the Services dashboard.

---

### Question 3. Vehicle Rental Jenkins Freestyle and webhook

**Question:** Q3. Jenkins Freestyle Project – Online Vehicle Rental Maven Java Application [25 Marks]

An online vehicle rental company frequently updates its Java standalone application with new vehicle details, rental charges, customer registration, and booking functionalities. The development team maintains the Maven Java project source code in a GitHub repository.  
The company wants to automate the application build process using a Jenkins Freestyle Project. Configure Jenkins to retrieve the source code from GitHub, execute Maven build operations, generate an executable JAR file, and run the application.  
Tasks:  
Create a new Freestyle Project in Jenkins named Vehicle-Rental.  
Clone the GitHub repository. https://github.com/haleema91/LAB-INTERNAL-II-OVR1.git  
Execute the following Maven build operations:  
Clean,install,test  
Execute the Jenkins build and verify the Console Output and show pipeline view.  
Configure GitHub Webhooks to trigger the Jenkins Freestyle Project automatically whenever developers push code changes.

**One-line explanation:** Fetch the project, build it in a Freestyle job, create Pipeline View and configure GitHub hook trigger for GITScm polling.

**Solution:**

**Tools:** Manage Jenkins → Plugins: Git, Maven Integration, Pipeline/Stage View as needed. Manage Jenkins → Tools: add JDK named `JDK21` and Maven named `MAVEN-HOME` using their installation folders.

1. **Create project:** Dashboard → New Item → `Vehicle-Rental` → Freestyle project → OK; select JDK `JDK21`.

2. **Clone repository:** Source Code Management → Git → URL `https://github.com/haleema91/LAB-INTERNAL-II-OVR1.git` → Branch `*/main`.

3. **Maven operations:** Add Invoke top-level Maven targets → Maven `MAVEN-HOME` → Goals `clean install test`. Then add Execute Windows batch command to make and run the executable JAR:

   ```cmd
   @echo off
   jar --update --file target\OVR-1-0.0.1-SNAPSHOT.jar --main-class KMIT.OVR_1.App
   if errorlevel 1 exit /b 1
   (echo Demo Customer&echo 1&echo 2) > rental-input.txt
   java -jar target\OVR-1-0.0.1-SNAPSHOT.jar < rental-input.txt
   ```


4. **Build and view:** Archive the artifacts → `target/*.jar`; publish JUnit report `target/surefire-reports/*.xml`. Save → Build Now → Console Output. Install Build Pipeline → Dashboard + → Build Pipeline View → Initial Job `Vehicle-Rental` → Save.

5. **Webhook:** use the following settings for automatic push-triggered builds.

**GitHub webhook settings:**

1. In **Vehicle-Rental → Configure → Build Triggers**, tick **GitHub hook trigger for GITScm polling**; leave Poll SCM/Build periodically off.

2. In CMD run `ngrok http 8080` and keep it open. If needed, first install ngrok and run `ngrok config add-authtoken YOUR_TOKEN`.

3. In **your own lab repo → Settings → Webhooks → Add webhook**, set Payload URL `https://YOUR_NGROK_DOMAIN/github-webhook/`, content type `application/json`, push event, Active → Add webhook. Use a repo you own and change the Jenkins checkout URL to it; leave the paper’s original repo unchanged.

4. Run one initial build, then push a harmless change to your lab repo. Check Webhook Recent Deliveries, the automatically started Jenkins build and Console Output; capture the screenshots requested by the paper.

**Scenario-based questions:**

- **The Jenkins build has completed successfully and generated the executable JAR file inside the target directory. How will you configure the Post-Build Action to archive the generated JAR file? Which file pattern should you specify?** Add Post-build Action → Archive the artifacts with `target/*.jar`, keep empty archives disallowed, and archive successful builds.
- **Developers have added new vehicle registration and booking features to GitHub. The company wants Jenkins to automatically execute the Freestyle Project whenever developers push changes. How will you configure the GitHub webhook and the appropriate build trigger in Jenkins?** In an authorized repository add a push webhook ending `/github-webhook/`, and tick Jenkins’s GitHub hook trigger for GITScm polling with no timed polling schedule.
- **The Jenkins Freestyle Project fails during Maven execution, displaying errors related to Java or Maven configuration. How will you identify the issue, verify the configured JDK and Maven versions, and troubleshoot the failed build using Console Output?** Read the first error in Console Output, verify Jenkins JDK/Maven tool names and paths, run version checks on the agent and confirm Java 17-or-newer compilation support.

---

### Question 4. WordPress and MySQL using Docker Compose

**Question:** Q4. Docker Compose – Online Vehicle Rental Website Using WordPress and MySQL [25 Marks]

An online vehicle rental company wants to develop a website where customers can view available vehicles, rental prices, and booking information. The company wants to use WordPress for website development and MySQL for database management.  
Create a Docker Compose file to run WordPress and MySQL in separate containers.  
Tasks:  
Create a Docker Compose YAML file.  
Configure WordPress and MySQL containers.  
Configure port mappings and persistent volumes.  
Configure environment variables for database connectivity.  
Start both containers using docker compose.  
Verify running containers using Docker Compose commands.  
Access the WordPress website through a web browser.

**One-line explanation:** Run WordPress and MySQL with database connectivity and persistent storage.

**Solution:**

1. Open **Command Prompt (CMD)** and create the working folder.

   ```cmd
   mkdir "%USERPROFILE%\se-lab-work\set8\q4"
   cd /d "%USERPROFILE%\se-lab-work\set8\q4"
   ```


2. Save the following as **docker-compose.yml**.

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


3. Start the services and check their status.

   ```cmd
   docker compose config --quiet
   docker compose up -d --build
   docker compose ps
   docker compose logs --tail 30
   ```


4. Open **http://localhost:8088**. Select language → Continue → enter site title, admin username/password and email → Install WordPress → Log In.

5. Publish a test post. Recreate the containers and confirm it remains; **do not use `down -v`**.

   ```cmd
   docker compose down
   docker compose up -d
   ```


**Scenario-based questions:**

- **You are running the Online Vehicle Rental application using Java and MySQL containers. After removing and recreating the MySQL container, you notice that all vehicle details, customer registrations, and booking information are lost. Why does this happen, and how can you configure Docker Compose volumes to preserve database data?** The data was in the deleted container layer; declare a named volume and mount it at `/var/lib/mysql`, retaining the volume across container recreation.
- **Your Online Vehicle Rental Java application is deployed using Docker Compose. However, the application fails to connect to the MySQL database. How do you identify whether both containers are running, check their logs.** Use `docker compose ps` and `docker compose logs web db` (or `wordpress db` for this stack), then verify shared network, service-name host and matching database credentials.
- **Your Maven Java standalone application is successfully packaged into a JAR file, but the Docker container exits immediately after execution. How will you identify the issue using Docker commands, inspect container logs and troubleshoot application execution errors?** Inspect `docker ps -a`, logs and exit code: exit 0 may mean normal task completion, while missing stdin, an absent main manifest or Java exceptions need the appropriate input/entry-point fix.

