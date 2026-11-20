# SET-6 — Solutions

Original question paper: `SET-6.docx` in MORNING.

**Commands:** Use Windows **CMD** locally. Use `bash` blocks only after SSH into Ubuntu EC2. Save code blocks in the named files.

---

### Question 1. Tomcat and MySQL using Docker Compose

**Question:** Q1.Docker Compose – Online Boutique Store (25 Marks)

An online boutique store wants to launch an e-commerce website. The application consists of a mysql backend and an tomcat web server. Create a Docker Compose file to run both applications in separate containers, exposing the mysql backend on port 3306 and tomcat on port 8080.  
Tasks:  
Create a Docker Compose YAML file.  
Configure appropriate port mappings and persistent volumes.  
Start both containers using docker compose  
Verify running containers using docker compose command.  
Access the tomcat server through a web browser.

**One-line explanation:** Run the boutique application on Tomcat with MySQL using the requested host ports.

**Solution:**

1. Open **Command Prompt (CMD)** and create the working folder.

   ```cmd
   mkdir "%USERPROFILE%\se-lab-work\set6\q1"
   cd /d "%USERPROFILE%\se-lab-work\set6\q1"
   ```


2. Clone the source into `app`, then save the following as **Dockerfile** beside `docker-compose.yml`.

   ```cmd
   git clone --branch main https://github.com/haleema91/LAB-INTERNAL-II-OBS.git app
   ```


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


3. Save the following as **docker-compose.yml**.

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


4. Start the services and check their status. Stop the local Jenkins service first because this question requires Tomcat on host port 8080.

   ```cmd
   docker compose config --quiet
   docker compose up -d --build
   docker compose ps
   docker compose logs --tail 30
   ```


5. Open **http://localhost:8080/OBS-1/**. Confirm the application page loads.

6. Verify MySQL; enter `LabBoutiquePass`, run `SHOW DATABASES;`, then `exit`. Its named volume preserves database files.

   ```cmd
   docker compose exec db mysql -uboutiqueuser -p boutique
   ```


**Note:** The supplied boutique page has no MySQL/JDBC implementation; sharing the network does not add one. Restart Jenkins after stopping this Compose stack.

**Scenario-based questions:**

- **Your Online Boutique Store application runs on Tomcat using port 8080, and MySQL uses port 3306. You want to access Tomcat through the browser and allow the application to communicate with MySQL. How do you configure port mappings and a custom Docker network in Docker Compose?** Publish Tomcat `8080:8080` and MySQL `127.0.0.1:3306:3306`, attach both to one custom network, and use `db:3306` for application database connections.
- **You are running an Online Boutique Store using Tomcat and MySQL containers. After removing and recreating the MySQL container, you notice that all customer registration and product details are lost. Why does this happen, and how can you configure Docker Compose volumes to preserve database data?** Data was kept only in the removed container’s writable layer; mount a named volume such as `mysql-data:/var/lib/mysql` and retain it when recreating the database.
- **Your Online Boutique Store is deployed using Tomcat and MySQL containers. When you access the application through http://localhost:8080, you receive a Connection Refused error. How do you identify whether the Tomcat container is running, check its logs, verify port mappings, troubleshoot the database connection, and safely stop and remove the containers?** Check `docker compose ps`, Tomcat/DB logs, `8080:8080` and database settings; use `/OBS-1/`, then safely remove the stack with `docker compose down` without `-v`.

---

### Question 2. Jenkins Pipeline, Tomcat and webhook

**Question:** Q2. Jenkins Pipeline for a Maven Web Project with web hooks[25 Marks]

The Online Boutique Store development team frequently pushes code changes to GitHub. The company wants to trigger the Jenkins Scripted Pipeline automatically whenever developers commit and push changes to the repository. Configure a GitHub webhook and the appropriate Jenkins build trigger to automatically execute the pipeline. The pipeline should perform GitHub clone, Maven clean, build, test, and package operations.  
Tasks:  
1. Clone Git Hub repository.( https://github.com/haleema91/LAB-INTERNAL-II-OBS.git)  
2. Configure Tomcat web server.  
3. Deploy the application on the Web Server  
4. Create the Pipeline View using Jenkins and show how code changes triggers jenkins using webhooks.

**One-line explanation:** Use a Scripted Pipeline to check out, build, test and package the project and deploy its WAR to Tomcat.

**Solution:**

**Tools:** Manage Jenkins → Plugins: Git, Maven Integration, Pipeline/Stage View as needed. Manage Jenkins → Tools: add JDK named `JDK21` and Maven named `MAVEN-HOME` using their installation folders.

1. **Clone:** the Checkout stage below uses `https://github.com/haleema91/LAB-INTERNAL-II-OBS.git` on branch `main`.

2. **Configure Tomcat:** Extract Tomcat 9 so `C:\tomcat9\bin` and `webapps` exist. Edit `conf\server.xml`: change the HTTP Connector from 8080 to **8085**. Start it with `C:\tomcat9\bin\startup.bat`; allow the Jenkins account to write to `C:\tomcat9\webapps`.

3. **Build and deploy:** New Item → `Set6_Build` → Pipeline → Pipeline script → paste this script → Save. Install Pipeline: Stage View if missing.

   ```groovy
   node {
       def jdk = tool 'JDK21'
       def mvn = tool 'MAVEN-HOME'
       withEnv(["JAVA_HOME=${jdk}", "PATH+JDK=${jdk}/bin", "PATH+MAVEN=${mvn}/bin"]) {
           try {
               stage('Clean workspace') { deleteDir() }
               dir('app') {
                   stage('Checkout') { git branch: 'main', url: 'https://github.com/haleema91/LAB-INTERNAL-II-OBS.git' }
                   stage('Clean') { bat 'call mvn -B clean' }
                   stage('Build') { bat 'call mvn -B compile' }
                   stage('Test') { bat 'call mvn -B test' }
                   stage('Package') { bat 'call mvn -B package' }
                   stage('Archive') { archiveArtifacts 'target/*.war' }
                   stage('Deploy') { bat 'copy /Y target\\OBS-1.war C:\\tomcat9\\webapps\\OBS-1.war' }
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


4. **Pipeline View and trigger:** Build Now → Console Output; check Stage View and `http://localhost:8085/OBS-1/`. Configure the webhook below.

**GitHub webhook settings:**

1. In **Set6_Build → Configure → Build Triggers**, tick **GitHub hook trigger for GITScm polling**; leave Poll SCM/Build periodically off.

2. In CMD run `ngrok http 8080` and keep it open. If needed, first install ngrok and run `ngrok config add-authtoken YOUR_TOKEN`.

3. In **your own lab repo → Settings → Webhooks → Add webhook**, set Payload URL `https://YOUR_NGROK_DOMAIN/github-webhook/`, content type `application/json`, push event, Active → Add webhook. Use a repo you own and change the Jenkins checkout URL to it; leave the paper’s original repo unchanged.

4. Run one initial build, then push a harmless change to your lab repo. Check Webhook Recent Deliveries, the automatically started Jenkins build and Console Output; capture the screenshots requested by the paper.

**Note:** These web repositories have no Java tests; “No tests to run” is not a passed test suite. A failed Maven command stops later stages. 

**Scenario-based questions:**

- **Your project manager asks you to remove previously generated build files before building the Online Boutique Store Maven Java Web Application. Which Maven command should you use in the pipeline?** Use `mvn clean` to remove previous Maven build output such as `target/` before rebuilding.
- **Your Online Boutique Store source code is available in a private GitHub repository. Where should you provide the repository URL in the Jenkins pipeline?** Provide the URL in the Pipeline’s `git` or `checkout` step with a stored `credentialsId`; never place a private GitHub password/token directly in the script.
- **Your Online Boutique Store application is building successfully, but the WAR file is not generated. Which Maven command should you execute in pipeline code?** Run `mvn package` (or `mvn clean package`) and confirm the POM uses `war` packaging; this repository generates `target/OBS-1.war`.

---

### Question 3. Kubernetes scaling and Nagios

**Question:** Q3. Kubernetes – Container Deployment and Auto Scaling and monitoring

[25 MARKS]  
During festival seasons, discount sales, and special promotional offers, thousands of customers access the Online Boutique Store simultaneously to browse products, check prices, and place orders. The company wants to deploy multiple application replicas using Kubernetes to handle increased customer traffic and ensure automatic recovery when application containers fail and monitor application availability using Nagios.  
Tasks:  
Start a Kubernetes cluster using Minikube.  
Create a Kubernetes Deployment nginx with two replicas.  
Expose the nginx using a Kubernetes Service of type NodePort at port 7070.  
Scale the deployment from two to five replicas to handle increased customer traffic.  
Verify the deployment, pods, services using appropriate Kubernetes commands.  
Install and configure Nagios Core using Docker to monitor the Online Boutique Store application.

**One-line explanation:** Deploy 2 replicas, expose the application on port 7070, scale to 5 and check it with Nagios.

**Solution:**

1. **Start Minikube:** open Docker Desktop first, then run in CMD.

   ```cmd
   minikube start --driver=docker
   minikube status
   kubectl config use-context minikube
   kubectl get nodes
   ```


2. **Create the Nginx Deployment with 2 replicas.**

   ```cmd
   kubectl create deployment set6-nginx --image=nginx:stable --replicas=2
   kubectl rollout status deployment/set6-nginx
   ```


3. **Expose the Service on port 7070:** container port is 80; Kubernetes allocates a valid NodePort.

   ```cmd
   kubectl expose deployment set6-nginx --type=NodePort --port=7070 --target-port=80
   kubectl get services
   minikube service set6-nginx --url
   ```


4. **Open in browser:** use the printed URL, or keep the following command running and open `http://localhost:7070/`. The all-interface binding lets Nagios reach this forwarded port.

   ```cmd
   kubectl port-forward --address=0.0.0.0 service/set6-nginx 7070:7070
   ```


5. **Scale to 5 replicas and verify** in a second CMD window.

   ```cmd
   kubectl scale deployment/set6-nginx --replicas=5
   kubectl rollout status deployment/set6-nginx
   kubectl get deployments,pods,services
   kubectl get pods -o wide
   ```


**Nagios monitoring:** Keep the port-forward CMD window open.

1. **Run Nagios** from CMD; stop an earlier Nagios container if port 8888 is occupied.

   ```cmd
   docker pull jasonrivers/nagios:latest
   docker run -d --name set6-nagios -p 8888:80 jasonrivers/nagios:latest
   docker ps
   ```


2. **Open dashboard:** `http://localhost:8888` → login `nagiosadmin` / `nagios` → Hosts and Services.

3. **Configure monitoring:** save this as `lab.cfg`. The application must be available on host port 7070.

   ```text
   define command {
       command_name check_lab_http
       command_line $USER1$/check_http -H host.docker.internal -p 7070 -u / -w 2 -c 5
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
   docker cp lab.cfg set6-nagios:/opt/nagios/etc/conf.d/lab.cfg
   docker exec set6-nagios /opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
   docker restart set6-nagios
   docker exec set6-nagios pgrep -a nagios
   ```


5. **Verify monitoring:** Services → Application HTTP → wait/force a check. Stop the application or its port-forward command and observe failure; restore it and confirm OK. These are dashboard alerts; email requires separate mail/contact configuration.

**Note:** This image uses `/opt/nagios`. `host.docker.internal` reaches the Windows host; `localhost` inside the container would check Nagios itself.

**Expected result:** 5 ready replicas and an accessible application URL. If a Pod is Pending, run `kubectl describe pod POD_NAME` and `kubectl get events`; inspect scheduling/resource errors.

**Scenario-based questions:**

- **Your Online Boutique Store is running successfully inside Minikube, but customers cannot access it through a web browser. How will you expose the application externally using a NodePort Service and access it through the Minikube service URL?** Expose Service port 7070 targeting Nginx 80 and open `minikube service set6-nginx --url` or forward `7070:7070`.
- **The boutique store administrator wants to confirm whether Nagios is running properly and monitoring the application server. Which commands will you execute to verify Nagios service status and configuration?** Check `docker ps`, logs, `docker exec set6-nagios pgrep -a nagios`, and `/opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg`, then inspect Services.
- **During a festival sale, you scale the Online Boutique Store deployment from two to five replicas. However, some Pods remain in the Pending state. What could be the possible reasons, and how would you troubleshoot them?** Check insufficient CPU/memory, scheduling rules and storage constraints with `kubectl describe pod`, `kubectl describe node` and `kubectl get events`.

---

### Question 4. Nginx static website on AWS EC2

**Question:** Q4. AWS EC2 Deployment – Online Boutique Store [25 Marks]

An online boutique company wants to launch its website on AWS. Create an Ubuntu EC2 instance, connect to it using SSH, install Docker, and deploy the website's index.html file using an Nginx Docker container. Verify the website using the EC2 public IPv4 address.  
Task: Deploy the online boutique store on EC2 instance.

**One-line explanation:** Deploy a static boutique page in Nginx on Ubuntu EC2.

**Solution:**

1. **Launch EC2:** AWS Console → EC2 → Launch instances → Ubuntu 24.04 x86-64 → lab-approved small instance (`t2.micro` if required/available). Create/download `set6-key.pem`. Enable public IP in a public subnet; Security Group inbound **SSH 22 → My IP; HTTP 80 → My IP or the required public audience**. Launch and copy Public IPv4. Free Tier depends on your account.

2. **Connect from CMD** in the folder containing your key; replace `PUBLIC_IP`.

   ```cmd
   cd /d "%USERPROFILE%\Downloads"
   ssh -i set6-key.pem ubuntu@PUBLIC_IP
   ```


3. **Inside Ubuntu EC2:** install the tools.

   ```bash
   sudo apt-get update
   sudo apt-get install -y docker.io git openjdk-21-jdk
   sudo systemctl enable --now docker
   ```


4. **Create the website:** run `mkdir -p ~/boutique-site && cd ~/boutique-site`. Save the first block as **index.html**, and the second as **Dockerfile** using `nano`; save with Ctrl+O, Enter, Ctrl+X.

   ```html
   <!doctype html>
   <html><body><h1>Online Boutique Store</h1><p>Welcome to our EC2 website.</p></body></html>
   ```


   ```dockerfile
   FROM nginx:stable
   COPY index.html /usr/share/nginx/html/index.html
   EXPOSE 80
   ```


5. **Build, run and check:** wait for web-server startup.

   ```bash
   sudo docker build -t set6-website:1.0 .
   sudo docker run -d --name set6-website --restart unless-stopped -p 80:80 set6-website:1.0
   sudo docker ps
   sudo docker logs --tail 30 set6-website
   curl -I http://localhost:80/
   ```


6. **Open in your laptop browser:** `http://PUBLIC_IP/`. If local curl works but the browser fails, check inbound port 80, the current public IP and routing/firewall.

**Scenario-based questions:**

- **The Online Boutique Store administrator has successfully launched an Ubuntu EC2 instance but needs to access it remotely to configure the website. How will you connect to the EC2 instance from your local machine using an SSH key pair? What steps will you follow if the SSH connection fails?** Run `ssh -i KEY.pem ubuntu@PUBLIC_IP`; if it fails, check instance/public IP, Security Group 22/My IP, username/key pair, routing and private-key file permissions.
- **An online boutique company wants to launch its shopping website on AWS. The administrator needs a Linux server to host the website. How will you launch an Ubuntu EC2 instance, select an appropriate instance type, configure a key pair, and create a Security Group to allow SSH and HTTP traffic?** Use EC2 Launch instances → Ubuntu x86-64 → an allowed small instance → downloaded `.pem` key → public subnet/public IP → Security Group SSH 22/My IP and HTTP 80.
- **The Online Boutique Store Docker container is running successfully on the AWS EC2 instance, but customers are unable to access the website through a browser. How will you identify the network issue, configure the EC2 Security Group to allow inbound HTTP traffic on port 80, and verify the website using the EC2 public IPv4 address?** Verify `curl localhost`, the container’s `80:80` mapping, inbound HTTP 80, public subnet/Internet Gateway and firewall, then browse `http://PUBLIC_IP/`.

