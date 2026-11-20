# SET-3 — Solutions

Original question paper: `Set-3.docx` in MORNING.

**Commands:** Use Windows **CMD** locally. Use `bash` blocks only after SSH into Ubuntu EC2. Save code blocks in the named files.

---

### Question 1. Flask using Docker Compose

**Question:** Q1. You are appointed as a DevOps engineer in a software development company. The development team wants to deploy a simple Python Flask application using Docker Compose. The application should run inside a Docker container, and developers should be able to modify the application code and rebuild it easily during development. [20 Marks]

**One-line explanation:** Build a Flask image and manage it using Docker Compose.

**Solution:**

1. Open **Command Prompt (CMD)** and create the working folder.

   ```cmd
   mkdir "%USERPROFILE%\se-lab-work\set3\q1"
   cd /d "%USERPROFILE%\se-lab-work\set3\q1"
   ```


2. Save **app.py**, **requirements.txt** and **Dockerfile** with these contents, in that order.

   ```python
   import os
   from flask import Flask
   app = Flask(__name__)

   @app.get("/")
   def home():
       return os.getenv("MESSAGE", "Hello from Flask + Docker!")

   if __name__ == "__main__":
       app.run(host="0.0.0.0", port=5000)
   ```


   ```text
   Flask==3.1.2
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
   name: set3-flask
   services:
     web:
       build: .
       ports:
         - "5000:5000"
   ```


4. Start the services and check their status.

   ```cmd
   docker compose config --quiet
   docker compose up -d --build
   docker compose ps
   docker compose logs --tail 30
   ```


5. Open **http://localhost:5000**. Confirm the application page loads.

6. For a code change, edit `app.py` locally, save, then rebuild. Use `docker compose down` when finished.

   ```cmd
   docker compose up -d --build web
   ```


**Scenario-based questions:**

- **A developer has deployed a Flask application displaying "Hello from Flask + Docker!". The developer wants to change the message to "Hello Docker Compose!". Modify the Python code to display the updated message. [ 1 Mark ]** Change the route’s default return value to `"Hello Docker Compose!"` (or set `MESSAGE` to that value in the supplied environment-configurable app).
- **A developer wants to access the Flask application through port 8080 on the host machine instead of port 5000. Modify the Docker Compose configuration to achieve this. [ 1 Mark ]** Change the web service mapping to `ports: ["8080:5000"]`; free host port 8080 first if local Jenkins uses it.
- **A software company wants to run two Flask applications simultaneously using Docker Compose. The first application should display "Welcome to Application 1", and the second should display "Welcome to Application 2". Task: Write a Docker Compose configuration to build and run two Flask services using the same Dockerfile but different host ports. [ 3 Marks]** Use `services: {app1: {build: ., environment: {MESSAGE: "Welcome to Application 1"}, ports: ["5001:5000"]}, app2: {build: ., environment: {MESSAGE: "Welcome to Application 2"}, ports: ["5002:5000"]}}` with the shared Dockerfile and environment-configurable `app.py` above.

---

### Question 2. Jenkins Scripted Pipeline

**Question:** Q2. A company needs working scripted pipeline (Groovy-based Jenkins file) that automates build and test stages successfully. Demonstrate this for project from GitHub with link https://github.com/anujyog1/carrepo.git. [25 Marks]

**One-line explanation:** Use a Scripted Pipeline to check out, build, test and package the project.

**Solution:**

**Tools:** Manage Jenkins → Plugins: Git, Maven Integration, Pipeline/Stage View as needed. Manage Jenkins → Tools: add JDK named `JDK21` and Maven named `MAVEN-HOME` using their installation folders.

1. New Item → `Set3_Build` → Pipeline → OK → Definition **Pipeline script**. Paste this Scripted Pipeline:

   ```groovy
   node {
       def jdk = tool 'JDK21'
       def mvn = tool 'MAVEN-HOME'
       withEnv(["JAVA_HOME=${jdk}", "PATH+JDK=${jdk}/bin", "PATH+MAVEN=${mvn}/bin"]) {
           try {
               stage('Clean workspace') { deleteDir() }
               dir('app') {
                   stage('Checkout') { git branch: 'master', url: 'https://github.com/anujyog1/carrepo.git' }
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

**Note:** These web repositories have no Java tests; “No tests to run” is not a passed test suite. A failed Maven command stops later stages. 

---

### Question 3. Email notifications and GitHub webhook

#### 3(a) Email notifications

**Question:** Q3. a. A client requires automatic email notifications on build success or failure of Jenkins scripted pipeline, constructed for project from GitHub with link https://github.com/anujyog1/carrepo.git. Demonstrate how you will configure Jenkins email notifications on successful and failed builds. [10 Marks]

**One-line explanation:** Send success/failure emails from the Scripted Pipeline in Question 2.

**Solution:**

1. Install **Email Extension**. Manage Jenkins → System → **Extended E-mail Notification** → enter your provider’s SMTP host, port, authentication credentials and TLS/SSL setting → Save. Keep the SMTP password in Jenkins credentials.

2. In Question 2’s Pipeline, add the following line inside `finally`, after the JUnit report line. Replace `YOUR_EMAIL` with your own address.

   ```groovy
   emailext to: 'YOUR_EMAIL',
       subject: "${env.JOB_NAME} #${env.BUILD_NUMBER}: ${currentBuild.currentResult}",
       body: "Result: ${currentBuild.currentResult}\n${env.BUILD_URL}"
   ```


3. Build normally to check the success email. Temporarily add `error('Notification test')` inside the Build stage to check the failure email; remove that line afterwards. These edits stay in Jenkins, not GitHub.

#### 3(b) GitHub webhook

**Question:** Demonstrate how Jenkins pipeline triggered automatically via GitHub Webhook without manual or SCM polling.        [15 Marks]

Expected Output:
- Webhook setup screenshot.
-  Illustration of auto trigger on code change in github
- Jenkins console output.

**One-line explanation:** Trigger Jenkins automatically from a GitHub push without a timed polling schedule.

**Solution:**

**GitHub webhook settings:**

1. In **Set3_Build → Configure → Build Triggers**, tick **GitHub hook trigger for GITScm polling**; leave Poll SCM/Build periodically off.

2. In CMD run `ngrok http 8080` and keep it open. If needed, first install ngrok and run `ngrok config add-authtoken YOUR_TOKEN`.

3. In **your own lab repo → Settings → Webhooks → Add webhook**, set Payload URL `https://YOUR_NGROK_DOMAIN/github-webhook/`, content type `application/json`, push event, Active → Add webhook. Use a repo you own and change the Jenkins checkout URL to it; leave the paper’s original repo unchanged.

4. Run one initial build, then push a harmless change to your lab repo. Check Webhook Recent Deliveries, the automatically started Jenkins build and Console Output; capture the screenshots requested by the paper.

---

### Question 4. Car application on AWS EC2

**Question:** Q4. As a team leader, help your teammate to deploy an application from github with link https://github.com/anujyog1/carrepo.git on an AWS EC2 Ubuntu instance. To show the deployment using public ip address, help him to launch EC2 instance, install docker and other required software’s, clone application, write the corresponding dockerfile, build the docker image and run it. [25 Marks]

**One-line explanation:** Deploy the application WAR on Ubuntu EC2 and open `/bookcar/` through the public IP.

**Solution:**

1. **Launch EC2:** AWS Console → EC2 → Launch instances → Ubuntu 24.04 x86-64 → lab-approved small instance (`t2.micro` if required/available). Create/download `set3-key.pem`. Enable public IP in a public subnet; Security Group inbound **SSH 22 → My IP; Custom TCP 8080 → My IP or the required public audience**. Launch and copy Public IPv4. Free Tier depends on your account.

2. **Connect from CMD** in the folder containing your key; replace `PUBLIC_IP`.

   ```cmd
   cd /d "%USERPROFILE%\Downloads"
   ssh -i set3-key.pem ubuntu@PUBLIC_IP
   ```


3. **Inside Ubuntu EC2:** install the tools and clone the repository.

   ```bash
   sudo apt-get update
   sudo apt-get install -y docker.io git openjdk-21-jdk
   sudo systemctl enable --now docker
   cd ~
   git clone --branch master https://github.com/anujyog1/carrepo.git
   cd carrepo
   ```


4. **Create Dockerfile:** run `nano Dockerfile`, paste the following, then Ctrl+O → Enter → Ctrl+X.

   ```dockerfile
   FROM maven:3.9-eclipse-temurin-21 AS build
   WORKDIR /app
   COPY pom.xml ./
   COPY src ./src
   RUN mvn -B clean package
   FROM tomcat:9.0-jdk21-temurin
   COPY --from=build /app/target/bookcar.war /usr/local/tomcat/webapps/bookcar.war
   EXPOSE 8080
   CMD ["catalina.sh", "run"]
   ```


5. **Build, run and check:** wait for web-server startup.

   ```bash
   sudo docker build -t set3-app:1.0 .
   sudo docker run -d --name set3-app --restart unless-stopped -p 8080:8080 set3-app:1.0
   sudo docker ps
   sudo docker logs --tail 30 set3-app
   curl -I http://localhost:8080/bookcar/
   ```


6. **Open in your laptop browser:** `http://PUBLIC_IP:8080/bookcar/`. If local curl works but the browser fails, check inbound port 8080, the current public IP and routing/firewall.

