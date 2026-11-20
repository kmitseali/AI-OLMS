# SET-2 — Solutions

Original question paper: `Set-2.docx` in MORNING.

**Commands:** Use Windows **CMD** locally. Use `bash` blocks only after SSH into Ubuntu EC2. Save code blocks in the named files.

---

### Question 1. Nginx and PostgreSQL using Docker Compose

**Question:** Q1. A software development company wants to deploy a simple application environment consisting of an Nginx web server and a PostgreSQL database. The administrator wants to manage both services using a single Docker Compose configuration file instead of creating and running containers individually. As a developer demonstrate how you will deploy it. [20 Marks]

**One-line explanation:** Run Nginx and PostgreSQL together using one Compose file.

**Solution:**

1. Open **Command Prompt (CMD)** and create the working folder.

   ```cmd
   mkdir "%USERPROFILE%\se-lab-work\set2\q1"
   cd /d "%USERPROFILE%\se-lab-work\set2\q1"
   ```


2. Save these files as **index.html** and **Dockerfile**, respectively.

   ```html
   <h1>Nginx and PostgreSQL Compose Lab</h1>
   ```


   ```dockerfile
   FROM nginx:stable
   COPY index.html /usr/share/nginx/html/index.html
   ```


3. Save the following as **docker-compose.yml**.

   ```yaml
   name: set2-compose

   services:
     web:
       build: .
       image: set2-web:1.0
       ports:
         - "8082:80"
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


5. Open **http://localhost:8082**. Confirm the application page loads.

6. Check PostgreSQL and demonstrate persistence; run the final query after the database is healthy again.

   ```cmd
   docker compose exec db psql -U labuser -d labdb -c "CREATE TABLE IF NOT EXISTS demo (id int); INSERT INTO demo VALUES (1);"
   docker compose down
   docker compose up -d
   docker compose ps
   docker compose exec db psql -U labuser -d labdb -c "SELECT * FROM demo;"
   ```


7. For a code change, edit `index.html` locally, save, then rebuild. Use `docker compose down` when finished.

   ```cmd
   docker compose up -d --build web
   ```


**Scenario-based questions:**

- **A developer is already running Nginx and PostgreSQL using Docker Compose. Now, the developer wants to add a Redis service. How can the existing Compose file be modified?** Add a `redis` service using `image: redis:7-alpine`, attach it to the same network, then run `docker compose up -d`; add a `/data` volume if its data must persist.
- **A web application requires a database to run properly. How can depends_on be used in Docker Compose to specify the startup dependency?** Set `depends_on: {db: {condition: service_healthy}}` on the web service and define a database healthcheck; short-form `depends_on: [db]` only orders startup.
- **An organization wants to run two instances of a web service to handle more requests. Which Docker Compose command can be used to scale the web service?** Run `docker compose up -d --scale web=2` after replacing the single fixed host port with distinct/dynamic ports or a reverse proxy and avoiding a fixed `container_name`.
- **A developer modifies the application code and wants to rebuild and restart the application using Docker Compose. Which command should be executed?** Run `docker compose up -d --build` to rebuild image-based application code and recreate affected services.
- **A developer wants to start, check, and stop multiple containers using Docker Compose. Which commands should be used?** Use `docker compose up -d` to start, `docker compose ps` to check, and `docker compose stop` to stop the stack.

---

### Question 2. Metro build, Pipeline View and Poll SCM

**Question:** Q2. You are appointed as a DevOps engineer for automating the build process of a web application using Maven and to perform following tasks: [25 Marks]

1. Pull the Maven web project from GitHub with link https://github.com/anujyog1/MetroRepo.git  
2. Build the project and create the Pipeline View using Jenkins.  
3. Demonstrate use of POLL SCM.

**One-line explanation:** Fetch the project, build it in a Freestyle job, create Pipeline View and configure Poll SCM.

**Solution:**

**Tools:** Manage Jenkins → Plugins: Git, Maven Integration, Pipeline/Stage View as needed. Manage Jenkins → Tools: add JDK named `JDK21` and Maven named `MAVEN-HOME` using their installation folders.

1. **Pull the project:** Jenkins → New Item → `Metro_Build` → Freestyle project → OK. Source Code Management → Git → URL `https://github.com/anujyog1/MetroRepo.git` → Branch `*/master`.

2. **Build and Pipeline View:** Add build step → Invoke top-level Maven targets → Maven `MAVEN-HOME` → Goals `clean package`. Post-build action → Archive the artifacts → `target/*.war`. Save → Build Now → Console Output. Install **Build Pipeline** → Dashboard **+ → Build Pipeline View** → Initial Job `Metro_Build` → Save.

3. **Poll SCM:** Configure → Build Triggers → tick **Poll SCM** → Schedule `H/5 * * * *` → Save. Jenkins builds only when the scheduled check finds a new commit; use Build Now for the initial build.

**Expected result:** `target/Book-Metro-Ticket.war`, BUILD SUCCESS and a successful build shown in Pipeline View. The inspected repository has no Java unit tests.

---

### Question 3. Nagios monitoring dashboard

**Question:** Q3. A start-up company wants dashboard displaying real-time status and health of the system in the company. Demonstrate which monitoring tool you will use to monitor CPU usage, memory, and service uptime through dashboard. [25 Marks]

**One-line explanation:** Use Nagios Core to display system metrics and application availability.

**Solution:**

1. **Run Nagios** from CMD; stop an earlier Nagios container if port 8888 is occupied.

   ```cmd
   docker pull jasonrivers/nagios:latest
   docker run -d --name set2-nagios -p 8888:80 jasonrivers/nagios:latest
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


4. **Configure monitoring:** save this as `lab.cfg`. The application must be available on host port 8082.

   ```text
   define command {
       command_name check_lab_http
       command_line $USER1$/check_http -H host.docker.internal -p 8082 -u / -w 2 -c 5
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
   docker cp check_lab_metrics.py set2-nagios:/opt/nagios/libexec/check_lab_metrics.py
   docker cp lab.cfg set2-nagios:/opt/nagios/etc/conf.d/lab.cfg
   docker exec set2-nagios /opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
   docker restart set2-nagios
   docker exec set2-nagios pgrep -a nagios
   ```


6. **Verify monitoring:** Services → Application HTTP → wait/force a check. Stop the application or its port-forward command and observe failure; restore it and confirm OK. These are dashboard alerts; email requires separate mail/contact configuration.

**Note:** These metric checks measure the Linux/Docker environment visible inside Nagios, not native Windows or every remote Pod. Use a target-side agent such as NCPA for remote host metrics.

**Note:** This image uses `/opt/nagios`. `host.docker.internal` reaches the Windows host; `localhost` inside the container would check Nagios itself.

---

### Question 4. Metro application on AWS EC2

**Question:** Q4. Your company wants you to deploy a application from github with link https://github.com/anujyog1/MetroRepo.git on an AWS EC2 Ubuntu instance. To show the deployment using public ip address, help your teammates to launch EC2, install docker and other required software’s, clone application, write the corresponding dockerfile, build the docker image and run it. [25 Marks]

**One-line explanation:** Deploy the application WAR on Ubuntu EC2 and open `/Book-Metro-Ticket/` through the public IP.

**Solution:**

1. **Launch EC2:** AWS Console → EC2 → Launch instances → Ubuntu 24.04 x86-64 → lab-approved small instance (`t2.micro` if required/available). Create/download `set2-key.pem`. Enable public IP in a public subnet; Security Group inbound **SSH 22 → My IP; Custom TCP 8080 → My IP or the required public audience**. Launch and copy Public IPv4. Free Tier depends on your account.

2. **Connect from CMD** in the folder containing your key; replace `PUBLIC_IP`.

   ```cmd
   cd /d "%USERPROFILE%\Downloads"
   ssh -i set2-key.pem ubuntu@PUBLIC_IP
   ```


3. **Inside Ubuntu EC2:** install the tools and clone the repository.

   ```bash
   sudo apt-get update
   sudo apt-get install -y docker.io git openjdk-21-jdk
   sudo systemctl enable --now docker
   cd ~
   git clone --branch master https://github.com/anujyog1/MetroRepo.git
   cd MetroRepo
   ```


4. **Create Dockerfile:** run `nano Dockerfile`, paste the following, then Ctrl+O → Enter → Ctrl+X.

   ```dockerfile
   FROM maven:3.9-eclipse-temurin-21 AS build
   WORKDIR /app
   COPY pom.xml ./
   COPY src ./src
   RUN mvn -B clean package
   FROM tomcat:9.0-jdk21-temurin
   COPY --from=build /app/target/Book-Metro-Ticket.war /usr/local/tomcat/webapps/Book-Metro-Ticket.war
   EXPOSE 8080
   CMD ["catalina.sh", "run"]
   ```


5. **Build, run and check:** wait for web-server startup.

   ```bash
   sudo docker build -t set2-app:1.0 .
   sudo docker run -d --name set2-app --restart unless-stopped -p 8080:8080 set2-app:1.0
   sudo docker ps
   sudo docker logs --tail 30 set2-app
   curl -I http://localhost:8080/Book-Metro-Ticket/
   ```


6. **Open in your laptop browser:** `http://PUBLIC_IP:8080/Book-Metro-Ticket/`. If local curl works but the browser fails, check inbound port 8080, the current public IP and routing/firewall.

