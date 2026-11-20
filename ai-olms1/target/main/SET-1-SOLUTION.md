# SET 1 Software Engineering Lab Internal II Solutions

**Original paper:** `Set-1.docx` in the MORNING folder.  
**Subject code in this paper:** 23CC505PC.  
**Format:** Question → One-line explanation → Solution → Expected result. Scenario questions have one-line answers.

This file answers the actual four questions in **Set-1.docx**. The pasted Library Management answer is used as the formatting reference; its six questions belong to a different paper.

## Quick navigation

- [Before starting](#before-starting)
- [Q1 WordPress and MySQL with Docker Compose](#q1-wordpress-and-mysql-with-docker-compose)
- [Q2 Maven build and Jenkins Pipeline View](#q2-maven-build-and-jenkins-pipeline-view)
- [Q3 Nginx with Minikube and four replicas](#q3-nginx-with-minikube-and-four-replicas)
- [Q4 Train application on AWS EC2](#q4-train-application-on-aws-ec2)
- [Final demonstration checklist](#final-demonstration-checklist)
- [Sources and verification](#sources-and-verification)

## Before starting

1. On Windows, open **Docker Desktop** and wait for its engine to start; use **Linux containers**.
2. Use **PowerShell** for local commands. Run commands labelled **Ubuntu terminal** only after connecting to EC2.
3. Paste YAML, XML, Java and Dockerfile blocks into the named files; these blocks are file contents, not terminal commands. Use spaces in YAML, not tabs.
4. In File Explorer, enable **View → Show → File name extensions** so files do not accidentally become `docker-compose.yml.txt` or `Dockerfile.txt`.
5. Keep practice files separate from these answer notes. The examples use `$HOME\se-lab-work\set1` on Windows and `~/trainrepo` on EC2.
6. Replace placeholders such as `PUBLIC_IP` and the key filename before running commands. Do not type angle brackets as part of a real IP address.

Check local tools:

```powershell
git --version
java -version
mvn -version
docker version
docker compose version
kubectl version --client
minikube version
```

If `minikube` is missing, install it using the [official Windows installation instructions](https://minikube.sigs.k8s.io/docs/start/), reopen PowerShell, and run `minikube version` again. Jenkins should already be installed and running for Q2; open [Jenkins](http://localhost:8080) and sign in with your existing account.

### Ports used in this set

| Component | Address or port |
|---|---|
| Jenkins on Windows | `http://localhost:8080` |
| WordPress on Windows | `http://localhost:8081` |
| Nginx through local port forwarding | `http://localhost:8090` |
| Train application on EC2 | `http://PUBLIC_IP:8080/booktrain/` |

Windows and EC2 are different computers, so Jenkins using local port 8080 does not conflict with port 8080 on EC2.

### Important findings from the question paper repositories

The repositories were inspected through fresh local clones on **20 November 2026**; no changes were sent to GitHub.

| Repository | Branch and inspected commit | What the solution must account for |
|---|---|---|
| [31octYogitaJava](https://github.com/anujyog1/31octYogitaJava) | `main`, `6e15817b594ae7281b352b9942b166f801767ef9` | Contains a README, a text file and `.gitignore`; **no `pom.xml` or Java source exists**. Q2 includes an explicitly labelled local demonstration project. |
| [trainrepo](https://github.com/anujyog1/trainrepo) | `master`, `dea73a8fef64ee59d49f60f96907281ee8808600` | Maven WAR project; output name `booktrain.war`, Java target 18, JSP pages, Servlet 4.0 descriptor; the POM has a stray UTF-8 BOM character **after** `</project>`. Q4 removes that character only in the EC2 clone. |

The train repository also contains an old `target/` directory. Q4 builds a fresh WAR from source rather than deploying that committed build output.

---

## Q1 WordPress and MySQL with Docker Compose

### Question

> You are assigned to deploy a WordPress website that requires a web server and a MySQL database. Instead of creating and managing containers individually, use Docker Compose to define and run both services through a single YAML configuration file. Configure the MySQL service using the mysql:5.7 image. Access the WordPress application through http://localhost:8081 and complete the initial WordPress setup. **[20 Marks]**
>
> Solve the given scenario based questions. **[5 × 1 marks = 5 Marks]**

**One-line explanation:** Define WordPress and MySQL 5.7 in one Compose file, connect them over a Docker network, preserve their data in named volumes, and open WordPress on host port 8081.

### Solution

#### Step 1 Create the working folder

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set1\q1-wordpress" | Out-Null
Set-Location "$HOME\se-lab-work\set1\q1-wordpress"
notepad docker-compose.yml
```

Click **Yes** if Notepad asks to create the file. Paste the following and save it.

#### Step 2 Write docker-compose.yml

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

**What the important settings do:**

- `mysql:5.7` matches the exact image requested in the paper. This is a legacy lab stack; the WordPress version is pinned to avoid silently moving to a future release requiring a newer database.
- `8081:80` means **Windows port 8081 → WordPress container port 80**.
- `db:3306` uses the Compose service name; `localhost` would point back to the WordPress container itself.
- Matching database names, usernames and passwords connect WordPress to MySQL. These are example lab passwords.
- `depends_on` with `service_healthy` waits for the database health check; short-form `depends_on` alone does not guarantee database readiness. [Docker startup order](https://docs.docker.com/compose/how-tos/startup-order/)
- The named volumes preserve the database and WordPress files when containers are recreated. No host port is published for MySQL.

#### Step 3 Validate and start the stack

Run these commands **from the folder containing `docker-compose.yml`**:

```powershell
docker compose config --quiet
docker compose pull
docker compose up -d
docker compose ps
```

Wait for `db` to become healthy and `wordpress` to be running. Initial database creation can take a minute or two.

To inspect startup messages:

```powershell
docker compose logs --tail 50 db
docker compose logs --tail 50 wordpress
```

#### Step 4 Complete the WordPress browser setup

1. Open [WordPress](http://localhost:8081).
2. Select **English** or another language → **Continue**.
3. Enter **Site Title**, for example `Set 1 WordPress Lab`.
4. Choose an administrator **Username**, **Password** and **Your Email**. These are WordPress login details, not the database credentials in YAML.
5. Click **Install WordPress** → **Log In**.
6. Sign in; the **Dashboard** should open.
7. Go to **Posts → Add New Post**, create a post titled `Persistence Test`, and publish it. This gives you data to check in Step 5.

#### Step 5 Verify data persistence

```powershell
docker compose down
docker compose up -d
docker compose ps
```

Wait for startup, reopen `http://localhost:8081`, and confirm that the site and `Persistence Test` post remain. `down` removes the containers and Compose network, while the named volumes remain. **Do not add `-v`** when you want to retain those volumes.

To pause the lab without removing containers:

```powershell
docker compose stop
```

To resume:

```powershell
docker compose start
```

#### If something does not work

| Problem | What to check |
|---|---|
| Cannot connect to the Docker engine | Open Docker Desktop and wait for the Linux engine to start. |
| `port is already allocated` | Check `docker ps`; stop the other application using host port 8081, because the paper requires this port. |
| Database connection error | Check `docker compose ps` and `docker compose logs db`; confirm `db:3306` and matching credentials. |
| Credentials were changed after the first run | MySQL initialization variables do not replace users/passwords in an existing database volume; use the original credentials or update the database account. |
| Image manifest error on an ARM computer | MySQL 5.7 is a legacy image; use an x86-64 lab PC or, with emulation available, add `platform: linux/amd64` to `db`. |

### Expected result

Both services run through Compose, WordPress opens at `http://localhost:8081`, setup completes, and the post remains after container recreation.

### Scenario-based questions and one-line answers

1. **A developer wants to check, view, remove and restart multiple containers using Docker Compose. Which commands should be used?**  
   Use `docker compose ps` to check status, `docker compose logs` to view output, `docker compose down` to remove the stack, and `docker compose restart` to restart it.

2. **An organization wants to protect its database records even after stopping or removing containers. How can a Docker Compose volume be configured?**  
   Declare `volumes: {mysql-data: {}}` at the top level and mount `mysql-data:/var/lib/mysql` under the MySQL service; retain that volume when removing containers.

3. **A another teammate wants to run an container using Docker Compose but wants to access it through port 8080 on the host machine. How can port mapping be configured?**  
   Set the service's `ports` to `["8080:80"]` when its application listens on container port 80, replacing 80 with the actual internal port if different.

4. **An organization wants to run two instances of a web service to handle more requests. Which Docker Compose command can be used to scale the web service?**  
   Use `docker compose up -d --scale web=2`; avoid a fixed `container_name` or shared fixed host port, and use distinct/dynamic published ports or a reverse proxy.

5. **You try to run two containers that both expose port 8080 on the host. What happens, and how can you run both apps?**  
   The second container fails to bind the same host IP and port; publish different host ports such as `8080:80` and `8082:80`, or route through a reverse proxy.

---

## Q2 Maven build and Jenkins Pipeline View

### Question

> You are appointed as a DevOps engineer for automating the build process of a simple Maven Java application using Maven and to perform following tasks: **[25 Marks]**
>
> 1. Pull the Java project from GitHub with link https://github.com/anujyog1/31octYogitaJava.git
> 2. Build the project and create the Pipeline View using Jenkins.
> 3. Demonstrate use of Build Periodically.

**One-line explanation:** Retrieve the repository, build a Maven application through linked Compile → Test → Package Freestyle jobs, display them in Build Pipeline View, and schedule the first job using Build Periodically.

### Repository issue you must know

At the inspected commit, the linked repository has **no Maven application**. Running `mvn clean package` directly in that checkout fails because `pom.xml` is missing. Changing the branch to `master` does not solve it; the repository's branch is `main`.

For an exact build of the intended application, obtain the missing Maven files or corrected repository URL from the examiner. The runnable demonstration below still checks out the required repository and then creates a **small local sample application in each Jenkins workspace**. It demonstrates the requested tools without pretending that these sample sources came from GitHub. No commit or push is required.

### Solution

#### Step 1 Configure Jenkins plugins and tools

1. Open [Jenkins](http://localhost:8080) and sign in.
2. Select **Manage Jenkins → Plugins → Available plugins**.
3. Install missing plugins: **Git**, **Maven Integration**, **JUnit**, and **Build Pipeline**. Restart Jenkins if prompted.
4. Open **Manage Jenkins → Tools**; older versions may call it **Global Tool Configuration**.
5. Under **JDK installations → Add JDK**, set:
   - **Name:** `JDK21`
   - Turn off **Install automatically** if using the existing JDK.
   - **JAVA_HOME:** the JDK folder containing `bin\java.exe`.
6. Under **Maven installations → Add Maven**, set:
   - **Name:** `MAVEN-HOME`
   - Turn off **Install automatically** if using the existing Maven installation.
   - **MAVEN_HOME:** the Maven folder containing `bin\mvn.cmd`.
7. Under **Git installations**, verify the Git executable, then click **Save**.

For the tools detected on this PC, the installation folders are:

```text
JAVA_HOME:   C:\Users\gayaz\Tools\jenkins\java\jdk-21.0.13+9
MAVEN_HOME:  C:\Users\gayaz\Tools\apache-maven-3.9.16
Git:         C:\Program Files\Git\cmd\git.exe
```

Use the actual paths if working on a different machine. The Jenkins service account must be able to read these installations and the helper script in Step 2.

#### Step 2 Prepare the missing-project demonstration helper

This helper is necessary **only because the supplied repository is incomplete**. It creates a sample `pom.xml`, one Java class and one real unit test in the destination folder.

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\se-lab-work\set1\q2-jenkins" | Out-Null
Set-Location "$HOME\se-lab-work\set1\q2-jenkins"
notepad prepare-demo.ps1
```

Paste the complete script below and save it. Here-string terminators `'@` must begin at the start of their lines.

**File: `prepare-demo.ps1`**

```powershell
param([string]$Destination = $env:WORKSPACE)
$ErrorActionPreference = 'Stop'
if ([string]::IsNullOrWhiteSpace($Destination)) {
    throw 'Pass -Destination with a project folder, or run this from Jenkins.'
}
New-Item -ItemType Directory -Force -Path $Destination | Out-Null
Set-Location -LiteralPath $Destination
New-Item -ItemType Directory -Force -Path 'src/main/java/lab' | Out-Null
New-Item -ItemType Directory -Force -Path 'src/test/java/lab' | Out-Null

@'
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>lab</groupId>
  <artifactId>set1-java-demo</artifactId>
  <version>1.0-SNAPSHOT</version>
  <packaging>jar</packaging>
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.release>17</maven.compiler.release>
  </properties>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.13.2</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.13.0</version>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>3.3.0</version>
      </plugin>
    </plugins>
  </build>
</project>
'@ | Set-Content -LiteralPath 'pom.xml' -Encoding ascii

@'
package lab;
public class App {
    public static int add(int a, int b) { return a + b; }
    public static void main(String[] args) {
        System.out.println("Hello from the Set 1 Maven demonstration");
    }
}
'@ | Set-Content -LiteralPath 'src/main/java/lab/App.java' -Encoding ascii

@'
package lab;
import org.junit.Test;
import static org.junit.Assert.assertEquals;
public class AppTest {
    @Test
    public void addsTwoNumbers() {
        assertEquals(5, App.add(2, 3));
    }
}
'@ | Set-Content -LiteralPath 'src/test/java/lab/AppTest.java' -Encoding ascii

Write-Host 'Created the local Maven demonstration; these files are not from GitHub.'
```

If the examiner supplies the real Maven project, omit this helper step and use that project's actual POM location instead.

#### Step 3 Create the Compile job

1. **Dashboard → New Item** → enter `Set1_Compile` → select **Freestyle project** → **OK**.
2. In **General**, select **JDK → JDK21** if the JDK selector is shown.
3. Open **Source Code Management → Git**:
   - **Repository URL:** `https://github.com/anujyog1/31octYogitaJava.git`
   - **Credentials:** none for this public repository.
   - **Branches to build → Branch Specifier:** `*/main`.
4. Under **Build Triggers**, tick **Build periodically** and enter:

   ```text
   H/5 * * * *
   ```

   This schedules the first job about every five minutes at Jenkins-assigned offsets, even when GitHub has not changed. Leave **Poll SCM** unticked for this question. [Jenkins cron syntax](https://www.jenkins.io/doc/book/pipeline/syntax/#jenkins-cron-syntax)

5. Under **Build Steps → Add build step → Execute Windows batch command**, enter:

   ```bat
   powershell.exe -NoProfile -ExecutionPolicy Bypass -File "C:\Users\gayaz\se-lab-work\set1\q2-jenkins\prepare-demo.ps1" -Destination "%WORKSPACE%"
   ```

   Change the helper path if you saved it elsewhere. `%WORKSPACE%` is Jenkins' own checkout folder; do not hard-code it.

6. Click **Add build step → Invoke top-level Maven targets**:
   - **Maven Version:** `MAVEN-HOME`
   - **Goals:** `clean compile`
   - Leave the POM field at its default `pom.xml`.
7. Click **Save**. The Git checkout happens before the helper and Maven steps.

#### Step 4 Create the Test and Package jobs

For each job, select **Dashboard → New Item**, enter the new name, and use **Copy from → Set1_Compile → OK**. Then change these settings:

| Setting | `Set1_Test` | `Set1_Package` |
|---|---|---|
| Build periodically | **Untick** | **Untick** |
| Maven goals | `clean test` | `clean package` |
| Git URL, branch, JDK, helper step | Keep copied settings | Keep copied settings |
| Post-build action | Publish JUnit test result report | Publish JUnit test result report; Archive the artifacts |
| Test report XMLs | `target/surefire-reports/*.xml` | `target/surefire-reports/*.xml` |
| Files to archive | — | `target/*.jar` |

To add a report, choose **Post-build Actions → Add post-build action → Publish JUnit test result report**. For packaging, also choose **Archive the artifacts**, enter `target/*.jar`, and select **Archive artifacts only if build is successful** if that option is shown. Leave **Allow empty archive** unticked. Save both jobs.

Each Freestyle job has a separate workspace, so each checks out and prepares its own project; it must not depend on another job's unshared `target/` folder. Maven lifecycle goals repeat earlier phases: `test` also compiles, and `package` also compiles and tests. This small lab chain makes the stages visible; for production, pass one fixed source revision through the chain or use a single Pipeline workspace. [Maven lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)

#### Step 5 Link the jobs in order

1. Open **Set1_Compile → Configure → Post-build Actions**.
2. Choose **Add post-build action → Build other projects**.
3. Enter `Set1_Test` and select **Trigger only if build is stable** → **Save**.
4. Open **Set1_Test → Configure → Post-build Actions → Build other projects**.
5. Enter `Set1_Package` and select **Trigger only if build is stable** → **Save**.
6. Do not add a downstream job to `Set1_Package`.

The resulting chain is:

```text
Set1_Compile → Set1_Test → Set1_Package
```

#### Step 6 Create the requested Build Pipeline View

1. Return to the **Dashboard**.
2. Click **+** beside the view tabs, or choose **New View**.
3. Enter **View name:** `Set1_Java_Pipeline`.
4. Select **Build Pipeline View** → **Create** or **OK**.
5. Under **Pipeline Flow**, select **Select Initial Job** if prompted.
6. Set **Initial Job:** `Set1_Compile`.
7. Set the number of displayed builds to `5` → **Apply → Save**.

The three linked Freestyle jobs should appear in order. This is the **Build Pipeline** plugin's view, which supports Freestyle job chains; a Pipeline job's Stage View uses a different plugin. [Build Pipeline plugin](https://plugins.jenkins.io/build-pipeline-plugin/)

#### Step 7 Build and verify

1. Open **Set1_Compile → Build Now** to demonstrate the first run immediately.
2. Click its build number under **Build History → Console Output**.
3. Confirm the Git checkout and Maven `BUILD SUCCESS`.
4. Return to `Set1_Java_Pipeline`; wait for **Test**, then **Package**, to complete.
5. Open **Set1_Test → latest build → Test Result**; confirm one test passed.
6. Open **Set1_Package → latest build**; confirm the artifact `set1-java-demo-1.0-SNAPSHOT.jar` is listed.
7. Wait for the next scheduled Compile build. Its build page should indicate **Started by timer**; its success should start the downstream jobs again.

To run the downloaded JAR from a Windows folder:

```powershell
java -cp .\set1-java-demo-1.0-SNAPSHOT.jar lab.App
```

This ordinary JAR has no `Main-Class` manifest, so use the classpath command above rather than `java -jar`.

#### If something does not work

| Problem | What to check |
|---|---|
| Missing POM | Confirm the helper ran before Maven; this error is expected from the unmodified supplied repository. |
| `Couldn't find any revision to build` | Set the branch to `*/main`, and check the Git URL. |
| Helper script not found or access denied | Correct its full path and grant the Jenkins service account read access; use a local path visible to that account. |
| Maven not found or invalid Java home | Recheck **Manage Jenkins → Tools**, the selected Maven version and JDK paths. |
| Plugin or dependency download failure | Read Console Output and check the Jenkins machine's internet/proxy access to Maven Central. |
| No Build Pipeline View option | Install **Build Pipeline** and restart if required. |
| Only one box in the view | Check the upstream jobs' **Build other projects** settings and select `Set1_Compile` as Initial Job. |
| Timer does not start builds | Confirm the job is enabled, **Build periodically** is ticked on Compile, and Jenkins is running. |

### Expected result

The three jobs succeed, the unit test is reported, the JAR is archived, and Build Pipeline View shows Compile → Test → Package. A later run starts automatically with **Started by timer**. Explain that the Maven sample is a local fallback for the incomplete GitHub repository.

After demonstrating scheduling, untick **Build periodically** on `Set1_Compile` and save if you no longer need repeated lab builds.

---

## Q3 Nginx with Minikube and four replicas

### Question

> Suppose your team wants to test Kubernetes setup locally using Minikube to create an Nginx deployment, expose service on port 8090, wants to verify in browser and list all pods and services. Help them to run Nginx service and Scale the Nginx deployment to 4 replicas. **[25 Marks]**

**One-line explanation:** Start Minikube, create an Nginx Deployment and a Service on port 8090 targeting Nginx port 80, forward local port 8090 to the Service, then scale to four replicas.

### Solution

#### Step 1 Start Minikube

Open **Docker Desktop**, then run in **Windows PowerShell**:

```powershell
minikube start --driver=docker
minikube status
kubectl config current-context
kubectl get nodes
```

The context should be `minikube` and the node should be `Ready`. If another context is selected:

```powershell
kubectl config use-context minikube
```

#### Step 2 Create the Nginx Deployment

```powershell
kubectl create deployment set1-nginx --image=nginx:stable --replicas=2
kubectl rollout status deployment/set1-nginx --timeout=180s
kubectl get deployments
kubectl get pods -l app=set1-nginx
```

Start with two replicas so that scaling to four is visible. Both Pods should become `Running` and `1/1` ready.

#### Step 3 Expose the Service on port 8090

```powershell
kubectl expose deployment set1-nginx --name=set1-nginx --type=NodePort --port=8090 --target-port=80
kubectl get services
kubectl describe service set1-nginx
```

Understand the three ports:

| Setting | Meaning |
|---|---|
| `port: 8090` | Port on the Kubernetes Service. |
| `targetPort: 80` | Port where Nginx listens inside each Pod. |
| Allocated `nodePort`, for example 31xxx | Port exposed on the Minikube node. It is normally allocated from 30000–32767. |

Do **not** set `--target-port=8090`; the standard Nginx image listens on 80. Do not set `nodePort: 8090`, because it is outside the usual NodePort range. [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)

#### Step 4 Open the application at localhost port 8090

Run:

```powershell
kubectl port-forward service/set1-nginx 8090:8090
```

Keep this terminal running. It should show a line similar to `Forwarding from 127.0.0.1:8090 -> 80`. Open [Nginx](http://localhost:8090) in your browser; the **Welcome to nginx!** page should appear.

If port 8090 is already in use, stop the other local process so the requested browser port is available.

For an alternative NodePort access demonstration, open another PowerShell window and run:

```powershell
minikube service set1-nginx --url
```

Open the URL it prints. On Windows with the Docker driver, keep that command's terminal open when it creates a tunnel; the printed local port can differ from 8090. [Minikube application access](https://minikube.sigs.k8s.io/docs/handbook/accessing/)

#### Step 5 Scale the Deployment to four replicas

Leave port forwarding running and use a **second PowerShell window**:

```powershell
kubectl scale deployment/set1-nginx --replicas=4
kubectl rollout status deployment/set1-nginx --timeout=180s
kubectl get deployment set1-nginx
kubectl get pods -l app=set1-nginx -o wide
kubectl get services
```

The Deployment should show `READY 4/4`, and the Pod list should show four ready Pods. Refresh the browser to confirm Nginx is still accessible.

#### Step 6 List all Pods and Services

```powershell
kubectl get pods
kubectl get services
```

These list resources in the current namespace. To include system namespaces as well:

```powershell
kubectl get pods --all-namespaces
kubectl get services --all-namespaces
```

#### If something does not work

```powershell
kubectl describe deployment set1-nginx
kubectl describe pods -l app=set1-nginx
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl logs deployment/set1-nginx --tail=50
```

- `Pending`: inspect scheduling events for insufficient CPU or memory.
- `ImagePullBackOff`: check internet access and the Nginx image name/tag.
- Browser connection refused: keep port forwarding running and confirm port 8090 is free.
- Service has no ready backends: compare the Service selector `app=set1-nginx` with the Pod labels and check Pod readiness.

### Expected result

Minikube is running, the Service exposes port 8090 and targets port 80, `http://localhost:8090` displays Nginx, and the Deployment has four ready replicas.

When finished, press **Ctrl+C** in the port-forward terminal. To remove only this question's resources:

```powershell
kubectl delete service set1-nginx
kubectl delete deployment set1-nginx
```

---

## Q4 Train application on AWS EC2

### Question

> Your company wants you to deploy a application from github with link https://github.com/anujyog1/trainrepo.git on an AWS EC2 Ubuntu instance. To show the deployment using public ip address, help your teammates to launch EC2, install docker and other required software’s, clone application, write the corresponding dockerfile, build the docker image and run it. **[25 Marks]**

**One-line explanation:** Launch an Ubuntu EC2 instance, clone the train WAR project, remove its stray POM character locally, build it using Maven in Docker, and serve it through Tomcat at the instance's public IP.

### Solution

#### Step 1 Launch the Ubuntu EC2 instance

1. Sign in to the [AWS Console](https://console.aws.amazon.com/) and select your lab region.
2. Search for **EC2** → **Instances → Launch instances**.
3. Set **Name:** `set1-train-app`.
4. Under **Application and OS Images**, choose **Ubuntu Server 24.04 LTS**, **64-bit x86**.
5. Choose a small compatible instance type, such as `t3.micro`, according to your lab/account allowance; use more memory if Maven builds run out of memory. Free Tier eligibility depends on the account and selected options. [AWS EC2 launch guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html)
6. Under **Key pair → Create new key pair**:
   - **Name:** `set1-train-key`
   - **Type:** RSA
   - **Private key format:** `.pem`
   - Click **Create key pair** and retain the downloaded file.
7. Under **Network settings → Edit**:
   - Use a public subnet with a route to an Internet Gateway.
   - **Auto-assign public IP:** Enable.
   - Create a Security Group named `set1-train-sg`.
   - Add the inbound rules shown below.
8. Leave sufficient root storage for Maven and Docker images; launch the instance.
9. Wait until it is **Running** and all instance status checks pass. Select it and copy **Public IPv4 address** from **Details**.

| Type | Port | Source | Purpose |
|---|---|---|---|
| SSH | 22 | My IP | SSH from your laptop. |
| Custom TCP | 8080 | My IP for your demonstration; `0.0.0.0/0` only when public access is required | Browser access to the published application port. |

Port 80 is unnecessary for the `:8080` URL used here. Keep outbound access available so the instance can download packages, Git source, Docker images and Maven dependencies.

#### Step 2 Connect from Windows using the downloaded key

**Windows PowerShell:**

```powershell
Set-Location "$HOME\Downloads"
ssh -i .\set1-train-key.pem ubuntu@PUBLIC_IP
```

Use your actual key location and EC2 public IP. Type `yes` when accepting the host key for this newly launched instance.

If SSH specifically reports that the private key permissions are too open, restrict that file and try again:

```powershell
$keyPath = (Resolve-Path '.\set1-train-key.pem').Path
$keyAccount = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
icacls $keyPath /inheritance:r
icacls $keyPath /grant:r "${keyAccount}:(R)"
ssh -i .\set1-train-key.pem ubuntu@PUBLIC_IP
```

If connection times out, check the current public IP, the inbound SSH rule's **My IP**, and that the instance is in a public subnet. If you see `Permission denied (publickey)`, check the key pair and the Ubuntu username `ubuntu`.

#### Step 3 Install Docker and Git on EC2

The prompt should now be similar to `ubuntu@ip-...:~$`. All commands until Step 9 run in this **Ubuntu terminal**.

```bash
sudo apt-get update
sudo apt-get install -y docker.io git python3 curl
sudo systemctl enable --now docker
sudo docker --version
git --version
```

This solution uses `sudo docker`, so logging out to refresh Docker group membership is unnecessary. Maven and Java are supplied by the build image and Tomcat image.

#### Step 4 Clone and inspect the correct application

```bash
cd ~
git clone --branch master https://github.com/anujyog1/trainrepo.git
cd trainrepo
git branch --show-current
find src -type f
```

The branch is `master`. The important source files are:

```text
trainrepo/
├── pom.xml
└── src/main/webapp/
    ├── index.jsp
    ├── register.jsp
    ├── trainchart.jsp
    └── WEB-INF/web.xml
```

This is a JSP web application packaged as a **WAR**, not a Spring Boot executable JAR. The original POM specifies Java 18, so use a JDK new enough to compile that target; JDK 21 is suitable. The Servlet 4.0 descriptor matches Tomcat 9.

#### Step 5 Remove the stray POM character locally

At the inspected commit, `pom.xml` contains an extra BOM character immediately after `</project>`. That makes it invalid XML. Run this small correction in the EC2 clone:

```bash
python3 - <<'PY'
from pathlib import Path
import xml.etree.ElementTree as ET

p = Path('pom.xml')
text = p.read_text(encoding='utf-8-sig')
text = text.replace('\ufeff', '')
ET.fromstring(text)
p.write_text(text, encoding='utf-8')
print('pom.xml is now valid XML in this local clone.')
PY
```

If GitHub later fixes the POM, the command is still safe: it validates the XML and leaves its Maven settings intact. This change stays on EC2; do not commit or push it.

#### Step 6 Create the Dockerfile and build context exclusions

Create the **Dockerfile** in the repository root, beside `pom.xml`:

```bash
cat > Dockerfile <<'EOF'
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml ./
COPY src ./src
RUN mvn -B clean package

FROM tomcat:9.0-jdk21-temurin
COPY --from=build /app/target/booktrain.war /usr/local/tomcat/webapps/booktrain.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
EOF
```

Create **.dockerignore**:

```bash
cat > .dockerignore <<'EOF'
.git
target
.settings
.classpath
.project
EOF
```

The first stage compiles and packages fresh source. The second stage runs Tomcat with the new WAR. Tomcat runs in the foreground so the container remains alive. The WAR filename makes the context path `/booktrain/`. [Official Tomcat image](https://hub.docker.com/_/tomcat)

#### Step 7 Build the image

```bash
sudo docker build -t set1-train:1.0 .
sudo docker images set1-train
```

Keep the final `.`; it tells Docker to use the current directory as the build context. The build output should include Maven `BUILD SUCCESS` and successful image creation. If Maven reports `No tests to run`, that reflects this repository's lack of unit tests; it does not mean tests passed.

#### Step 8 Run the application container

```bash
sudo docker run -d --name set1-train --restart unless-stopped -p 8080:8080 set1-train:1.0
sudo docker ps --filter name=set1-train
sudo docker logs --tail 100 set1-train
curl -I http://localhost:8080/booktrain/
```

Wait for Tomcat startup and deployment messages. `docker ps` should show a host mapping such as `0.0.0.0:8080->8080/tcp`; the local HTTP check should return a successful response.

`EXPOSE 8080` documents the internal port; `-p 8080:8080` is what actually publishes that port on EC2. `--restart unless-stopped` allows Docker to restart the container after a Docker/instance restart, unless you explicitly stopped it.

#### Step 9 Verify through the public IP in your laptop browser

Open:

```text
http://PUBLIC_IP:8080/booktrain/
```

Replace `PUBLIC_IP` with the actual EC2 public IPv4 address. Use **http**, include **:8080**, and include **/booktrain/**.

Confirm that:

1. The home page displays **Welcome to the Indian Railway**.
2. **Passenger Registration** opens `register.jsp`.
3. Submitting a test name and email displays **Registration Successful!**.
4. **Train Chart** opens `trainchart.jsp` and shows the train details.

The inspected application only displays submitted registration details; it does not implement database-backed registration. Do not claim persistent ticket or user storage for this repository.

#### If something does not work

| Symptom | What to check or run |
|---|---|
| POM parse error | Repeat Step 5; inspect the file for content after `</project>`. |
| `invalid target release: 18` | Confirm the build uses `maven:3.9-eclipse-temurin-21`, not a Java 17 build image. |
| Build killed or exit code 137 | Check memory with `free -h`; use a larger instance or suitable swap space for the lab build. |
| Container is absent from `docker ps` | Run `sudo docker ps -a --filter name=set1-train` and inspect `sudo docker logs set1-train`. |
| HTTP 404 at `/` | Use `/booktrain/`; the official Tomcat image need not provide a root welcome application. |
| HTTP 404 at `/booktrain/` | Inspect Tomcat logs for WAR deployment errors and confirm the WAR filename and copy path. |
| Local `curl` works but laptop browser times out | Check EC2 Security Group inbound TCP 8080, the current public IP, public subnet routing and any Ubuntu firewall rules. |
| Port already allocated | Run `sudo docker ps` and `sudo ss -lntp`; stop the conflicting service or deliberately change both host mapping and browser port. |
| Container name already exists | Use `sudo docker start set1-train` for the existing container; to replace it after a new build, stop/remove that named container first. |

To confirm the deployed WAR is present:

```bash
sudo docker exec set1-train ls -l /usr/local/tomcat/webapps
```

### Expected result

The Ubuntu instance runs a detached Tomcat container built from the GitHub source, and `http://PUBLIC_IP:8080/booktrain/` displays the train application and its working JSP pages.

For lab cleanup, stop the container with `sudo docker stop set1-train`. After saving any needed work, use **EC2 → Instances → select the lab instance → Instance state → Terminate instance** when you are finished with the instance.

---

## Final demonstration checklist

| Question | Evidence to show |
|---|---|
| Q1 | Compose YAML with `mysql:5.7`; both services running; WordPress on `localhost:8081`; installation completed; post retained after recreation. |
| Q2 | Repository checkout on `main`; missing-project explanation; three successful linked jobs; one reported test; archived JAR; Build Pipeline View; timer-started run. |
| Q3 | Minikube node Ready; Nginx page on `localhost:8090`; Service port 8090 targeting 80; Deployment `4/4`; Pod and Service lists. |
| Q4 | EC2 instance and Security Group; SSH session; local POM correction; Dockerfile; fresh image build; running container; train pages at the public IP URL. |

## Sources and verification

Repository-specific observations above come from fresh local clones at the recorded commits. A cached web view of `trainrepo` showed an older, more severely broken POM; the instructions follow the **fresh clone**, whose remaining XML issue is the trailing BOM.

- Compose configuration and readiness: [Docker Compose startup order](https://docs.docker.com/compose/how-tos/startup-order/) and [WordPress official image](https://hub.docker.com/_/wordpress).
- Jenkins job-chain visualization: [Build Pipeline plugin](https://plugins.jenkins.io/build-pipeline-plugin/); timer expressions: [Jenkins cron syntax](https://www.jenkins.io/doc/book/pipeline/syntax/#jenkins-cron-syntax).
- Maven phases: [Maven build lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html).
- Kubernetes ports and local access: [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/) and [Minikube application access](https://minikube.sigs.k8s.io/docs/handbook/accessing/).
- EC2 launch and network settings: [AWS EC2 getting started](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html); container deployment: [Tomcat official image](https://hub.docker.com/_/tomcat).

**Validation limits:** The commands are study and execution instructions, not a claim that your Jenkins jobs, Kubernetes cluster or AWS instance have already been created. Repository inspection and local validation performed while preparing this answer are reported separately below.

### Checks completed while preparing this answer

| Check | Result |
|---|---|
| Fresh checkout of both GitHub repositories | Branches, source structure and recorded commits confirmed; both inspection clones remained unchanged. |
| Q1 Compose file | Passed `docker compose config --quiet`. |
| Docker image tags | WordPress `6.8.3-php8.2-apache`, MySQL `5.7`, Maven `3.9-eclipse-temurin-21` and Tomcat `9.0-jdk21-temurin` were available when checked. |
| Q2 helper script | Executed successfully and generated the documented Maven project. |
| Q2 Maven build | `mvn clean package` succeeded on JDK 21; one unit test passed with zero failures/errors; JAR generated. |
| Q2 application execution | The documented `java -cp ... lab.App` command printed the expected greeting. |
| Q4 POM correction and Maven build | Corrected XML validated; `mvn clean package` succeeded on JDK 21 and generated `booktrain.war`. |
| Markdown structure | All navigation links resolve to headings and all code fences are balanced. |

Full container startup, WordPress installation, Jenkins UI configuration, Minikube execution and AWS deployment were not performed while writing the answer; those remain the practical steps to demonstrate in your lab.
