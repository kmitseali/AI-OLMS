# SET-1 — Solutions

Original question paper: `Set-1.docx` in MORNING.

**Commands:** Use Windows **CMD** locally. Use `bash` blocks only after SSH into Ubuntu EC2. Save code blocks in the named files.

---

### Question 1. WordPress and MySQL using Docker Compose

**Question:** Q1. You are assigned to deploy a WordPress website that requires a web server and a MySQL database. Instead of creating and managing containers individually, use Docker Compose to define and run both services through a single YAML configuration file. Configure the MySQL service using the mysql:5.7 image. Access the WordPress application through http://localhost:8081 and complete the initial WordPress setup. [20 Marks]

**One-line explanation:** Run WordPress and MySQL 5.7 using Compose and complete setup on port 8081.

**Solution:**

1. Open **Command Prompt (CMD)** and create the working folder.

   ```cmd
   mkdir "%USERPROFILE%\se-lab-work\set1\q1"
   cd /d "%USERPROFILE%\se-lab-work\set1\q1"
   ```


2. Save the following as **docker-compose.yml**.

   ```yaml
   name: set1-wordpress

   services:
     db:
       image: mysql:5.7
       restart: unless-stopped
       environment:
         MYSQL_ROOT_PASSWORD: RootLab57Pass
         MYSQL_DATABASE: wordpress
         MYSQL_USER: wpuser
         MYSQL_PASSWORD: WpLab57Pass
       volumes:
         - mysql-data:/var/lib/mysql
       networks:
         - wp-net
       healthcheck:
         test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -p$$MYSQL_ROOT_PASSWORD --silent"]
         interval: 10s
         timeout: 5s
         retries: 12
         start_period: 30s

     wordpress:
       image: wordpress:6.8.3-php8.2-apache
       restart: unless-stopped
       depends_on:
         db:
           condition: service_healthy
       ports:
         - "8081:80"
       environment:
         WORDPRESS_DB_HOST: db:3306
         WORDPRESS_DB_NAME: wordpress
         WORDPRESS_DB_USER: wpuser
         WORDPRESS_DB_PASSWORD: WpLab57Pass
       volumes:
         - wordpress-data:/var/www/html
       networks:
         - wp-net

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


4. Open **http://localhost:8081**. Select language → Continue → enter site title, admin username/password and email → Install WordPress → Log In.

5. Publish a test post. Recreate the containers and confirm it remains; **do not use `down -v`**.

   ```cmd
   docker compose down
   docker compose up -d
   ```


**Scenario-based questions:**

- **A developer wants to check, view, remove and restart multiple containers using Docker Compose. Which commands should be used?** Use `docker compose ps` to check status, `docker compose logs` to view output, `docker compose down` to remove the stack, and `docker compose restart` to restart it.
- **An organization wants to protect its database records even after stopping or removing containers. How can a Docker Compose volume be configured?** Declare `volumes: {mysql-data: {}}` at the top level and mount `mysql-data:/var/lib/mysql` under the MySQL service; retain that volume when removing containers.
- **A another teammate wants to run an container using Docker Compose but wants to access it through port 8080 on the host machine. How can port mapping be configured?** Set the service's `ports` to `["8080:80"]` when its application listens on container port 80, replacing 80 with the actual internal port if different.
- **An organization wants to run two instances of a web service to handle more requests. Which Docker Compose command can be used to scale the web service?** Use `docker compose up -d --scale web=2`; avoid a fixed `container_name` or shared fixed host port, and use distinct/dynamic published ports or a reverse proxy.
- **You try to run two containers that both expose port 8080 on the host. What happens, and how can you run both apps?** The second container fails to bind the same host IP and port; publish different host ports such as `8080:80` and `8082:80`, or route through a reverse proxy.

---

### Question 2. Maven build, Pipeline View and Build Periodically

**Question:** Q2. You are appointed as a DevOps engineer for automating the build process of a simple Maven Java application using Maven and to perform following tasks: [25 Marks]

1. Pull the Java project from GitHub with link https://github.com/anujyog1/31octYogitaJava.git  
2. Build the project and create the Pipeline View using Jenkins.  
3. Demonstrate use of Build Periodically.

**One-line explanation:** Fetch the project, build it in a Freestyle job, create Pipeline View and configure Build periodically.

**Solution:**

**Tools:** Manage Jenkins → Plugins: Git, Maven Integration, Pipeline/Stage View as needed. Manage Jenkins → Tools: add JDK named `JDK21` and Maven named `MAVEN-HOME` using their installation folders.

1. **Pull the project:** Jenkins → New Item → `Java_Build` → Freestyle project → OK. Source Code Management → Git → URL `https://github.com/anujyog1/31octYogitaJava.git` → Branch `*/main`.

**Repository issue:** This URL contains no Maven project. Use the intended project if supplied; otherwise the following step creates a clearly labelled **local demo**, without changing GitHub.

2. **Build and Pipeline View:** add an Execute Windows batch command step to create the missing local demo once. Use your installed Maven path.

   ```cmd
   if not exist demo\pom.xml call "C:\Users\gayaz\Tools\apache-maven-3.9.16\bin\mvn.cmd" -B archetype:generate -DarchetypeGroupId=org.apache.maven.archetypes -DarchetypeArtifactId=maven-archetype-quickstart -DarchetypeVersion=1.5 -DgroupId=lab -DartifactId=demo -DinteractiveMode=false
   ```


   Add **Invoke top-level Maven targets** → Maven `MAVEN-HOME` → Goals `clean package` → Advanced → POM `demo/pom.xml`. Add **Archive the artifacts** → `demo/target/*.jar`. Save → Build Now → Console Output. Install **Build Pipeline** → Dashboard **+ → Build Pipeline View** → Initial Job `Java_Build` → Save.

3. **Build periodically:** Configure → Build Triggers → tick **Build periodically** → Schedule `H/5 * * * *` → Save. Wait for a build marked **Started by timer**, even if Git has not changed.

---

### Question 3. Nginx deployment and scaling with Minikube

**Question:** Q3. Suppose your team wants to test Kubernetes setup locally using Minikube to create an Nginx deployment, expose service on port 8090, wants to verify in browser and list all pods and services. Help them to run Nginx service and Scale the Nginx deployment to 4 replicas. [25 Marks]

**One-line explanation:** Deploy 2 replicas, expose the application on port 8090, scale to 4.

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
   kubectl create deployment set1-nginx --image=nginx:stable --replicas=2
   kubectl rollout status deployment/set1-nginx
   ```


3. **Expose the Service on port 8090:** container port is 80; Kubernetes allocates a valid NodePort.

   ```cmd
   kubectl expose deployment set1-nginx --type=NodePort --port=8090 --target-port=80
   kubectl get services
   minikube service set1-nginx --url
   ```


4. **Open in browser:** use the printed URL, or keep the following command running and open `http://localhost:8090/`.

   ```cmd
   kubectl port-forward service/set1-nginx 8090:8090
   ```


5. **Scale to 4 replicas and verify** in a second CMD window.

   ```cmd
   kubectl scale deployment/set1-nginx --replicas=4
   kubectl rollout status deployment/set1-nginx
   kubectl get deployments,pods,services
   kubectl get pods -o wide
   ```


**Expected result:** 4 ready replicas and an accessible application URL. If a Pod is Pending, run `kubectl describe pod POD_NAME` and `kubectl get events`; inspect scheduling/resource errors.

---

### Question 4. Train application on AWS EC2

**Question:** Q4. Your company wants you to deploy a application from github with link https://github.com/anujyog1/trainrepo.git on an AWS EC2 Ubuntu instance. To show the deployment using public ip address, help your teammates to launch EC2, install docker and other required software’s, clone application, write the corresponding dockerfile, build the docker image and run it. [25 Marks]

**One-line explanation:** Deploy the application WAR on Ubuntu EC2 and open `/booktrain/` through the public IP.

**Solution:**

1. **Launch EC2:** AWS Console → EC2 → Launch instances → Ubuntu 24.04 x86-64 → lab-approved small instance (`t2.micro` if required/available). Create/download `set1-key.pem`. Enable public IP in a public subnet; Security Group inbound **SSH 22 → My IP; Custom TCP 8080 → My IP or the required public audience**. Launch and copy Public IPv4. Free Tier depends on your account.

2. **Connect from CMD** in the folder containing your key; replace `PUBLIC_IP`.

   ```cmd
   cd /d "%USERPROFILE%\Downloads"
   ssh -i set1-key.pem ubuntu@PUBLIC_IP
   ```


3. **Inside Ubuntu EC2:** install the tools and clone the repository.

   ```bash
   sudo apt-get update
   sudo apt-get install -y docker.io git openjdk-21-jdk
   sudo systemctl enable --now docker
   cd ~
   git clone --branch master https://github.com/anujyog1/trainrepo.git
   cd trainrepo
   ```


   The inspected train POM contains a stray character after `</project>`; correct it **locally** before building:

   ```bash
   python3 -c "from pathlib import Path; p=Path('pom.xml'); p.write_text(p.read_text(encoding='utf-8-sig').replace(chr(65279),''),encoding='utf-8')"
   ```

4. **Create Dockerfile:** run `nano Dockerfile`, paste the following, then Ctrl+O → Enter → Ctrl+X.

   ```dockerfile
   FROM maven:3.9-eclipse-temurin-21 AS build
   WORKDIR /app
   COPY pom.xml ./
   COPY src ./src
   RUN mvn -B clean package
   FROM tomcat:9.0-jdk21-temurin
   COPY --from=build /app/target/booktrain.war /usr/local/tomcat/webapps/booktrain.war
   EXPOSE 8080
   CMD ["catalina.sh", "run"]
   ```


5. **Build, run and check:** wait for web-server startup.

   ```bash
   sudo docker build -t set1-app:1.0 .
   sudo docker run -d --name set1-app --restart unless-stopped -p 8080:8080 set1-app:1.0
   sudo docker ps
   sudo docker logs --tail 30 set1-app
   curl -I http://localhost:8080/booktrain/
   ```


6. **Open in your laptop browser:** `http://PUBLIC_IP:8080/booktrain/`. If local curl works but the browser fails, check inbound port 8080, the current public IP and routing/firewall.

