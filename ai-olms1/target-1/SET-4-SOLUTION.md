# SET-4 — Solutions

Original question paper: `SET-4.docx` in MORNING.

**Commands:** Use Windows **CMD** locally. Use `bash` blocks only after SSH into Ubuntu EC2. Save code blocks in the named files.

**Repository note:** The event repo builds `OEMS-1.war`; examples rename it to match `/OnlineEventManagement/`. Its `RegisterServlet` handler is absent.

---

### Question 1. Tomcat and Nginx using Docker Compose

**Question:** Q1. Docker Compose – Online Event Management System Using Tomcat and NGINX [25 Marks]

An organization wants to deploy an Online Event Management System using Docker Compose. The application is developed using Java (JSP and Servlets) and runs on an Apache Tomcat server. NGINX is used as a reverse proxy to handle incoming client requests and forward them to the Tomcat application server.  
The organization wants to use Docker Compose to manage both containers and establish communication between them.  
Tasks:  
Create a Docker Compose YAML file to run Apache Tomcat and NGINX in separate containers.  
Configure the Tomcat container to expose port 8080 and NGINX to expose port 80.  
Start both containers using the docker compose command.  
Verify the running containers, network, and services using Docker Compose commands.  
Access the Online Event Management application through the NGINX server using a web browser.

**One-line explanation:** Run the event application on Tomcat and access it through an Nginx reverse proxy.

**Solution:**

1. Open **Command Prompt (CMD)** and create the working folder.

   ```cmd
   mkdir "%USERPROFILE%\se-lab-work\set4\q1"
   cd /d "%USERPROFILE%\se-lab-work\set4\q1"
   ```


2. Clone the source into `app`, then save the following as **Dockerfile** beside `docker-compose.yml`.

   ```cmd
   git clone --branch main https://github.com/haleema91/LAB-INTERNAL-II-OEMS.git app
   ```


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


3. Save this as **nginx.conf**; requests retain the `/OnlineEventManagement/` path.

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


4. Save the following as **docker-compose.yml**.

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


5. Start the services and check their status. Stop the local Jenkins service first because this question requires Tomcat on host port 8080.

   ```cmd
   docker compose config --quiet
   docker compose up -d --build
   docker compose ps
   docker compose logs --tail 30
   ```


6. Open **http://localhost/OnlineEventManagement/**. Confirm the application page loads.

**Scenario-based questions:**

- **The NGINX container is unable to communicate with the Tomcat container. Explain how you will troubleshoot the issue by checking the custom Docker network, service names, container connectivity, and port configuration.** Check `docker network inspect set4-events_event-net`, both service memberships, `tomcat:8080`, and `docker compose exec nginx curl -I http://tomcat:8080/OnlineEventManagement/`.
- **The Tomcat container is running, but users receive a 502 Bad Gateway error when accessing the application through NGINX. Identify the possible causes and explain how you will troubleshoot the NGINX configuration, upstream server address, Tomcat status, and container logs.** Validate `nginx -t`, verify upstream `tomcat:8080`, and inspect `docker compose logs nginx tomcat` for connection refusal, slow startup or a failed Tomcat process.
- **Another application is already using port 80 on the host machine. Explain how you will modify the Docker Compose YAML file to expose NGINX on a different host port while keeping the internal NGINX port unchanged.** Change Nginx’s mapping from `"80:80"` to `"8082:80"` and browse `http://localhost:8082/OnlineEventManagement/` while keeping Nginx’s internal listen port 80.

---

### Question 2. Event application on AWS EC2

**Question:** Q2. AWS EC2 Deployment – Online Event Management Maven Web Application [25 Marks]

An event management company has developed a Java-based Online Event Management web application using Maven, JSP, and Servlets. The application allows customers to view upcoming events, register for events, and book tickets.  
The administrator wants to deploy this Maven web application on an AWS EC2 instance using Apache Tomcat. The application must be packaged as a WAR file using Maven and deployed on the Tomcat server running on Ubuntu EC2.  
Task: Deploy the online event management application on AWS using an EC2 instance.( https://github.com/haleema91/LAB-INTERNAL-II-OEMS.git)

**One-line explanation:** Deploy the application WAR on Ubuntu EC2 and open `/OnlineEventManagement/` through the public IP.

**Solution:**

1. **Launch EC2:** AWS Console → EC2 → Launch instances → Ubuntu 24.04 x86-64 → lab-approved small instance (`t2.micro` if required/available). Create/download `set4-key.pem`. Enable public IP in a public subnet; Security Group inbound **SSH 22 → My IP; Custom TCP 9090 → My IP or the required public audience**. Launch and copy Public IPv4. Free Tier depends on your account.

2. **Connect from CMD** in the folder containing your key; replace `PUBLIC_IP`.

   ```cmd
   cd /d "%USERPROFILE%\Downloads"
   ssh -i set4-key.pem ubuntu@PUBLIC_IP
   ```


3. **Inside Ubuntu EC2:** install the tools and clone the repository.

   ```bash
   sudo apt-get update
   sudo apt-get install -y docker.io git openjdk-21-jdk
   sudo systemctl enable --now docker
   cd ~
   git clone --branch main https://github.com/haleema91/LAB-INTERNAL-II-OEMS.git
   cd LAB-INTERNAL-II-OEMS
   ```


4. **Create Dockerfile:** run `nano Dockerfile`, paste the following, then Ctrl+O → Enter → Ctrl+X.

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


5. **Build, run and check:** wait for web-server startup.

   ```bash
   sudo docker build -t set4-app:1.0 .
   sudo docker run -d --name set4-app --restart unless-stopped -p 9090:8080 set4-app:1.0
   sudo docker ps
   sudo docker logs --tail 30 set4-app
   curl -I http://localhost:9090/OnlineEventManagement/
   ```


6. **Open in your laptop browser:** `http://PUBLIC_IP:9090/OnlineEventManagement/`. If local curl works but the browser fails, check inbound port 9090, the current public IP and routing/firewall.

**Scenario-based questions:**

- **The Tomcat server is running successfully on the EC2 instance, but users cannot access the Online Event Management application using the EC2 public IPv4 address. What changes are required in the EC2 Security Group to allow inbound traffic on port 9090? How will you verify the Tomcat port and troubleshoot connectivity issues?** Allow inbound TCP 9090 from the required client source, verify the host mapping `9090:8080`, and compare local curl, Docker port output and Tomcat logs before checking external routing/firewalls.
- **After deploying the WAR file, the administrator accesses the application using: http://<EC2-Public-IP>:9090/OnlineEventManagement/ The browser displays an HTTP 404 error. Identify the possible reasons and explain how you will verify WAR deployment, application naming, Tomcat logs, JSP files and servlet URL mappings.** Confirm the WAR was deliberately named `OnlineEventManagement.war`, verify its webapps deployment and JSP/servlet mappings, and read Tomcat logs; the repository’s original context is `/OEMS-1/`.
- **The administrator executes mvn clean package to generate the Online Event Management WAR file. However, the WAR file is not generated inside the target directory. Identify the possible causes and explain how you will troubleshoot the Maven project structure, pom.xml configuration, dependencies, and compilation errors.** Run Maven from the folder with `pom.xml`, confirm `<packaging>war</packaging>`, and fix the first dependency/compilation/plugin error in `mvn clean package` output before inspecting `target/`.

---

### Question 3. Jenkins Freestyle and Poll SCM

**Question:** Q3. Jenkins Freestyle Project – Maven Web Project [25 Marks]

An event management company has developed a Java-based Online Event Management System using Maven, JSP, and Servlets. The source code is maintained in a GitHub repository. The company wants to automate the application build process using a Jenkins Freestyle project.  
The Jenkins server must fetch the source code from GitHub, execute Maven build commands, generate a WAR file, and verify the successful build.  
Tasks:  
Create a new Jenkins Freestyle Project named OnlineEventManagement.  
Clone the GitHub repository(https://github.com/haleema91/LAB-INTERNAL-II-OEMS.git)  
Configure the appropriate Git branch (main/master).  
Configure the build trigger using POLL scm to execute the project manually or whenever changes are pushed to GitHub.  
Verify the build execution status using the Jenkins pipeline view.

**One-line explanation:** Fetch the project, build it in a Freestyle job, create Pipeline View and configure Poll SCM.

**Solution:**

**Tools:** Manage Jenkins → Plugins: Git, Maven Integration, Pipeline/Stage View as needed. Manage Jenkins → Tools: add JDK named `JDK21` and Maven named `MAVEN-HOME` using their installation folders.

1. **Create project:** Dashboard → New Item → `OnlineEventManagement` → Freestyle project → OK.

2. **Clone repository:** Source Code Management → Git → Repository URL `https://github.com/haleema91/LAB-INTERNAL-II-OEMS.git`.

3. **Select branch:** Branches to build → `*/main`.

4. **Configure Poll SCM:** Build Triggers → Poll SCM → `H/5 * * * *` → Save. Use Build Now for a manual run.

5. **Build and verify:** Add build step → Invoke top-level Maven targets → Maven `MAVEN-HOME` → Goals `clean package`. Add Archive the artifacts → `target/*.war`. Save → Build Now → Console Output; expect `target/OEMS-1.war`.

6. **Pipeline View:** install Build Pipeline → Dashboard + → Build Pipeline View → Initial Job `OnlineEventManagement` → Save. A successful build appears in the view.

**Scenario-based questions:**

- **The Jenkins Freestyle project is unable to fetch the Online Event Management source code from GitHub. Identify the possible causes and explain how you will troubleshoot the repository URL, Git installation, branch name, and GitHub credentials.** Verify the repository URL, installed Git executable, actual `main` branch, network access and any required stored Jenkins credential using Console Output.
- **The Jenkins build is successful in fetching the source code, but the WAR file is not generated in the target directory. Explain how you will troubleshoot the Maven packaging configuration, build goals, project structure, and Console Output.** Check the workspace POM, `war` packaging, `clean package` goals, actual `target/OEMS-1.war` name and the first Maven error in Console Output.
- **A developer modifies the event registration Servlet but introduces Java compilation errors. Jenkins reports the build as FAILED. Explain how you will identify the compilation errors using Console Output, correct the source code, and execute the build again.** Read the compiler’s filename/line in Console Output, correct the source in an authorized lab copy or local demonstration, and rebuild; the inspected repository currently contains no event registration Servlet.

---

### Question 4. Kubernetes deployment, scaling and Nagios

**Question:** Q4. Kubernetes – Application Deployment, Scaling and Monitoring [25 Marks]

During popular events such as concerts, conferences, and festivals, thousands of users access the Online Event Management System simultaneously. The company wants to deploy multiple application replicas using Kubernetes to handle increased traffic and ensure application availability.Nagios will be used to monitor the application server.  
Tasks:  
Start a Kubernetes cluster using Minikube.  
Create a Kubernetes Deployment Nginx with two replicas.  
Expose the deployment using a NodePort Service and expose the application on port 6060.  
Scale the deployment from two to three replicas.  
Verify the deployment, Pods and Services using appropriate Kubernetes commands.  
Install and configure Nagios Core using Docker to monitor the application server.

**One-line explanation:** Deploy 2 replicas, expose the application on port 6060, scale to 3 and check it with Nagios.

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
   kubectl create deployment set4-nginx --image=nginx:stable --replicas=2
   kubectl rollout status deployment/set4-nginx
   ```


3. **Expose the Service on port 6060:** container port is 80; Kubernetes allocates a valid NodePort.

   ```cmd
   kubectl expose deployment set4-nginx --type=NodePort --port=6060 --target-port=80
   kubectl get services
   minikube service set4-nginx --url
   ```


4. **Open in browser:** use the printed URL, or keep the following command running and open `http://localhost:6060/`. The all-interface binding lets Nagios reach this forwarded port.

   ```cmd
   kubectl port-forward --address=0.0.0.0 service/set4-nginx 6060:6060
   ```


5. **Scale to 3 replicas and verify** in a second CMD window.

   ```cmd
   kubectl scale deployment/set4-nginx --replicas=3
   kubectl rollout status deployment/set4-nginx
   kubectl get deployments,pods,services
   kubectl get pods -o wide
   ```


**Nagios monitoring:** Keep the port-forward CMD window open.

1. **Run Nagios** from CMD; stop an earlier Nagios container if port 8888 is occupied.

   ```cmd
   docker pull jasonrivers/nagios:latest
   docker run -d --name set4-nagios -p 8888:80 jasonrivers/nagios:latest
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
   docker cp lab.cfg set4-nagios:/opt/nagios/etc/conf.d/lab.cfg
   docker exec set4-nagios /opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
   docker restart set4-nagios
   docker exec set4-nagios pgrep -a nagios
   ```


5. **Verify monitoring:** Services → Application HTTP → wait/force a check. Stop the application or its port-forward command and observe failure; restore it and confirm OK. These are dashboard alerts; email requires separate mail/contact configuration.

**Note:** This image uses `/opt/nagios`. `host.docker.internal` reaches the Windows host; `localhost` inside the container would check Nagios itself.

**Expected result:** 3 ready replicas and an accessible application URL. If a Pod is Pending, run `kubectl describe pod POD_NAME` and `kubectl get events`; inspect scheduling/resource errors.

**Scenario-based questions:**

- **During a major concert ticket release, the company increases the application replicas from two to five. Some Pods remain in the Pending state. What are the possible reasons and which Kubernetes commands will you use to troubleshoot the issue?** Insufficient resources, scheduling constraints or unbound storage can prevent scheduling; inspect `kubectl describe pod POD`, `kubectl get events` and node capacity before adding capacity or correcting constraints.
- **A customer cannot access the Online Event Management application through a browser even though the Pods are running. How will you configure and verify the NodePort Service and access the application through Minikube?** Verify the Service selector and `targetPort: 80`, use Service port 6060 with an allocated valid NodePort, and open `minikube service set4-nginx --url` or forward `6060:6060`.
- **One of the application Pods crashes during online ticket booking. Explain how Kubernetes Deployment ensures that the required number of replicas is maintained.** Kubelet restarts a crashed container in its Pod; if the Pod is lost/deleted, the Deployment’s ReplicaSet creates a replacement to restore the desired replica count.
- **The administrator wants to verify whether Nagios is running and monitoring the application server. Which commands will you execute to check Nagios status and configuration?** Run `docker ps --filter name=set4-nagios`, `docker logs set4-nagios`, `docker exec set4-nagios pgrep -a nagios`, and the `/opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg` preflight check.

