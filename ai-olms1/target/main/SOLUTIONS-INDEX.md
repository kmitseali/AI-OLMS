# Software Engineering Lab Internal II Solutions

All **nine papers** have separate Markdown solution files directly in the MORNING folder. Open the Markdown file for the set you need; each is self-contained and includes the original practical questions, a one-line explanation, detailed execution steps, code/configuration, verification and troubleshooting. Scenario questions have one-line answers.

| Paper | Open solution | Main topics |
|---|---|---|
| Set 1 | [SET-1-SOLUTION.md](SET-1-SOLUTION.md) | WordPress/MySQL 5.7, Jenkins timer and Pipeline View, Nginx on Minikube, train app on EC2 |
| Set 2 | [SET-2-SOLUTION.md](SET-2-SOLUTION.md) | Nginx/PostgreSQL, Jenkins Poll SCM, Nagios, Metro app on EC2 |
| Set 3 | [SET-3-SOLUTION.md](SET-3-SOLUTION.md) | Flask Compose, Scripted Pipeline, email/webhook, car app on EC2 |
| Set 4 | [SET-4-SOLUTION.md](SET-4-SOLUTION.md) | Tomcat/Nginx reverse proxy, event app on EC2, Freestyle, Kubernetes/Nagios |
| Set 5 | [SET-5-SOLUTION.md](SET-5-SOLUTION.md) | Blood Bank Pipeline, EC2, Minikube/Nagios, web/PostgreSQL Compose |
| Set 6 | [SET-6-SOLUTION.md](SET-6-SOLUTION.md) | Boutique Tomcat/MySQL, webhook deployment, Kubernetes/Nagios, static Nginx on EC2 |
| Set 7 | [SET-7-SOLUTION.md](SET-7-SOLUTION.md) | Apartment CI/webhook, EC2 port 8090, Kubernetes/Nagios, Tomcat/PostgreSQL Compose |
| Set 8 | [SET-8-SOLUTION.md](SET-8-SOLUTION.md) | Vehicle Rental console app, Kubernetes/Nagios, Freestyle/webhook, WordPress/MySQL |
| Set 9 | [SET-9-SOLUTION.md](SET-9-SOLUTION.md) | Sports Scripted Pipeline, EC2, Kubernetes/Nagios, Flask/PostgreSQL/Redis |

## How to use the files

1. Open the desired set and use **Quick navigation** to jump to a question.
2. Read **Before starting** and the repository findings first; they identify the actual branch, build output and source limitations.
3. Run local commands in **Windows PowerShell** and EC2 commands in the **Ubuntu SSH terminal**. Save file-content blocks into the named files.
4. Follow the steps in order, then check the stated browser URL, logs, artifact or dashboard result.
5. Use the scenario answers directly; they intentionally do not repeat the execution walkthrough.

The original question papers are unchanged. GitHub repositories were inspected through read-only clones; no remote files, settings, commits or webhooks were changed. For a webhook/push demonstration, the guides use a lab repository you own or are authorized to administer.

## Important repository findings

- **Set 1 Java repository:** no Maven project is present at the inspected commit; the guide clearly labels its local demonstration fallback.
- **Set 1 train repository:** the POM has a stray character after the closing XML tag; the guide documents a local correction.
- **Most web repositories:** contain JSP pages with no Java unit tests; a successful package or “No tests to run” does not mean a test suite passed.
- **Missing backend handlers:** the Blood Bank `donor`, Apartment `tenant`, Events `RegisterServlet` and Sports `RegistrationServlet` endpoints are absent from the inspected source; their corresponding links/forms cannot become functional merely by deploying the WAR.
- **Event repository:** builds `OEMS-1.war`; the EC2/proxy examples deliberately rename it to match the paper’s `/OnlineEventManagement/` URL.
- **Vehicle Rental:** is an interactive console program, not a web server; the guide handles standard input, the missing Main-Class manifest and normal container exit.
- **Nagios:** the selected image uses `/opt/nagios`; local Docker metrics are distinguished from native Windows or remote-server metrics.

## Validation performed

All 36 practical questions and 110 scenario questions are covered. Each remaining paper’s source text was checked against its answer file. Repository branches, commits, POM XML, artifact names, main classes/JSP filenames and source-test presence were inspected. All nine complete Compose configurations passed `docker compose config --quiet`; embedded YAML, Python syntax, PowerShell syntax, Markdown links and code fences were checked.

Set 1’s documented Maven examples were built successfully, including its one passing demonstration test. Sets 2–9 were checked statically: automatic approval review rejected the batch Maven run because it would execute third-party build code across multiple repositories with network access. No end-to-end Jenkins, email, webhook, AWS or Kubernetes deployment is claimed. Each set records these limits so expected results are not confused with completed lab runs.
