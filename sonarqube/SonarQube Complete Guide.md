# SonarQube: Complete Theoretical & Hands-On Demo Guide — Installation, Jenkins Integration & Scenarios

## Executive Summary

SonarQube is an open-source platform for continuous inspection of code quality. It performs static code analysis to detect bugs, vulnerabilities, code smells, security hotspots, and duplications across 30+ programming languages. This guide covers SonarQube theory, types of issues, installation on an AWS EC2 instance, integration with Jenkins, and multiple demo scenarios with complete commands and execution steps.[^1][^2]

***

## Part 1: Theoretical Concepts

### What is SonarQube?

SonarQube is a static code analysis tool developed by SonarSource. It continuously inspects source code **without executing it**, looking for patterns that indicate bugs, security vulnerabilities, and poor coding practices. It is language-agnostic and supports Java, Python, JavaScript, TypeScript, Go, C#, C/C++, Ruby, Kotlin, PHP, and many more.[^1]

SonarQube operates on a **client-server model**:

- **SonarQube Server** — The central hub that stores analysis results, quality profiles, quality gates, and the web UI dashboard. It runs on port `9000` by default.[^3]
- **SonarScanner** — The client-side agent that performs the actual code analysis and pushes results to the SonarQube Server.
- **Database** — PostgreSQL (recommended), Microsoft SQL Server, or Oracle to persist project data, metrics, and rules.[^3]

### Why SonarQube Matters in DevOps

In a CI/CD pipeline, code quality is often an afterthought. SonarQube solves this by acting as an automated code reviewer that runs on every build. The key use cases include:

- **Continuous Code Inspection** — Automatically analyze code on every commit or PR merge.[^1]
- **Security Vulnerability Detection** — Identify SQL injections, XSS, hardcoded credentials, and other OWASP Top 10 issues.[^2]
- **Technical Debt Management** — Calculate the effort (in person-days) needed to fix all issues, visualized as "technical debt".[^1]
- **Code Review & Collaboration** — Developers see issues directly in the dashboard with line-level precision.[^1]
- **Quality Gates** — Enforce release-readiness criteria; if code doesn't meet thresholds, the CI/CD pipeline fails automatically.[^4]
- **Customizable Quality Profiles** — Tailor rule sets per language and project to match organizational coding standards.[^1]

### Types of Issues Detected by SonarQube

SonarQube categorizes detected issues into several types:[^5][^6]

| Issue Type | Description | Example |
|---|---|---|
| **Bug** | Code that will produce an error or unexpected behavior at runtime | Null pointer dereference, infinite loop, resource leak |
| **Vulnerability** | Security flaw that can be exploited by an attacker | SQL injection, XSS, hardcoded passwords |
| **Code Smell** | Code that works but is poorly written, causing maintainability problems | Methods exceeding 80 lines, unused variables, deeply nested conditions |
| **Security Hotspot** | Code that needs manual review to determine if it is a real vulnerability | Use of cryptographic APIs, cookie handling, regex patterns |
| **Duplication** | Repeated blocks of code across the project | Copy-pasted functions, duplicated logic |
| **Coverage Gap** | Code not covered by unit tests | Untested branches, methods without assertions |

### Issue Severity Levels

Each issue is assigned a severity:[^7][^6]

| Severity | Impact | Example |
|---|---|---|
| **Blocker** | High probability of application crash or data corruption | Memory leak, unclosed JDBC connection |
| **Critical** | High probability of impacting application behavior or represents a security flaw | SQL injection, empty catch block |
| **Major** | Quality flaw that highly impacts developer productivity | Uncovered code, duplicated blocks, unused parameters |
| **Minor** | Quality flaw that slightly impacts developer productivity | Lines too long, naming convention violations |
| **Info** | Neither a bug nor a quality flaw, informational only | TODO comments in code |

### Quality Profiles

A Quality Profile is a collection of rules that SonarQube uses to analyze code for a specific language. Each language has a default "Sonar way" profile built-in. Organizations can:[^1]

- Create custom quality profiles per project or team
- Activate or deactivate individual rules
- Set rule severity levels
- Inherit from parent profiles to build layered rule sets

### Quality Gates

A Quality Gate is a pass/fail mechanism that determines whether a project is ready for release. It consists of conditions like:[^8][^4]

- No new blocker or critical issues
- Code coverage on new code must be > 80%
- Duplicated lines on new code must be < 3%
- Security rating must be A

If any condition fails, the Quality Gate status is **Failed**, and the CI/CD pipeline can be configured to abort.[^9][^4]

### Clean Code Attributes (New Taxonomy)

Starting from SonarQube 10.x, issues are also classified using "Clean Code" attributes that map to software qualities:[^6]

- **Consistency** — Code follows a uniform style
- **Intentionality** — Code clearly expresses its purpose
- **Adaptability** — Code is easy to change
- **Responsibility** — Code handles errors and edge cases properly

### SonarQube Editions

| Edition | Key Features |
|---|---|
| **Community (Free)** | 15+ languages, basic analysis, quality gates, quality profiles |
| **Developer** | Branch analysis, PR decoration, 27+ languages |
| **Enterprise** | Portfolio management, executive reporting, governance |
| **Data Center** | High availability, horizontal scaling for large deployments |

***

## Part 2: Installation on EC2 (Ubuntu) — Complete Demo

### Prerequisites

- AWS EC2 instance: **t2.medium** or larger (minimum 2 vCPU, 4 GB RAM, 30 GB disk)[^3]
- OS: **Ubuntu 22.04 LTS**
- Security Group: Open ports **9000** (SonarQube), **22** (SSH), **8080** (Jenkins)
- A key pair for SSH access

### Step 1: Launch EC2 Instance

```bash
# Launch via AWS Console or CLI
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t2.medium \
  --key-name your-key-pair \
  --security-group-ids sg-xxxxxxxx \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=SonarQube-Server}]'
```

After launch, SSH into the instance:

```bash
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>
```

### Step 2: Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### Step 3: Install Java 17

SonarQube requires Java 17 (since version 9.9+):[^10][^3]

```bash
sudo apt install openjdk-17-jdk -y

# Verify installation
java -version
# Expected: openjdk version "17.x.x"
```

### Step 4: Install and Configure PostgreSQL

SonarQube needs a relational database. PostgreSQL is the recommended choice:[^3]

```bash
# Install PostgreSQL
sudo apt install curl ca-certificates -y
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc \
  --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc

sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] \
  https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
  > /etc/apt/sources.list.d/pgdg.list'

sudo apt update
sudo apt install postgresql-15 -y
```

Create the SonarQube database and user:

```bash
sudo -i -u postgres

# Inside postgres shell
createuser sonar
createdb sonar -O sonar
psql

# Inside psql prompt
ALTER USER sonar WITH ENCRYPTED PASSWORD 'StrongP@ss123';
\q

# Exit postgres user
exit
```

### Step 5: Configure System Limits

SonarQube requires elevated system limits:[^3]

```bash
# Set vm.max_map_count
sudo sysctl -w vm.max_map_count=524288
sudo sysctl -w fs.file-max=131072

# Make changes permanent
sudo bash -c 'cat >> /etc/sysctl.conf <<EOF
vm.max_map_count=524288
fs.file-max=131072
EOF'

sudo sysctl -p

# Set ulimits for sonarqube user
sudo bash -c 'cat >> /etc/security/limits.conf <<EOF
sonarqube   -   nofile   131072
sonarqube   -   nproc    8192
EOF'
```

### Step 6: Download and Install SonarQube

```bash
# Download SonarQube Community Edition (latest LTS)
cd /opt
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.5.1.90531.zip

# Install unzip if needed
sudo apt install unzip -y

# Extract
sudo unzip sonarqube-10.5.1.90531.zip
sudo mv sonarqube-10.5.1.90531 /opt/sonarqube

# Create dedicated user
sudo adduser --system --no-create-home --group --disabled-login sonarqube

# Set ownership
sudo chown -R sonarqube:sonarqube /opt/sonarqube
```

### Step 7: Configure SonarQube

Edit the configuration file to connect to PostgreSQL:[^3]

```bash
sudo nano /opt/sonarqube/conf/sonar.properties
```

Uncomment and update these lines:

```properties
sonar.jdbc.username=sonar
sonar.jdbc.password=StrongP@ss123
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonar

# Web server binding
sonar.web.host=0.0.0.0
sonar.web.port=9000
```

### Step 8: Create Systemd Service

```bash
sudo nano /etc/systemd/system/sonarqube.service
```

Add the following content:[^3]

```ini
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
User=sonarqube
Group=sonarqube
Restart=always
LimitNOFILE=131072
LimitNPROC=8192

[Install]
WantedBy=multi-user.target
```

Start and enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl start sonarqube
sudo systemctl enable sonarqube

# Check status
sudo systemctl status sonarqube

# View logs if needed
tail -f /opt/sonarqube/logs/sonar.log
```

### Step 9: Access SonarQube UI

Open browser and navigate to:

```
http://<EC2-PUBLIC-IP>:9000
```

Default credentials:[^11][^3]

- **Username:** `admin`
- **Password:** `admin`

On first login, SonarQube will prompt to change the default password.

### Alternative: Install SonarQube Using Docker Compose

For a quicker setup, Docker Compose can be used:[^12]

```bash
# Install Docker and Docker Compose
sudo apt update
sudo apt install docker.io docker-compose -y
sudo systemctl start docker
sudo systemctl enable docker

# Set vm.max_map_count (required for Elasticsearch inside SonarQube)
sudo sysctl -w vm.max_map_count=524288
echo "vm.max_map_count=524288" | sudo tee -a /etc/sysctl.conf
```

Create `docker-compose.yml`:

```yaml
version: "3"
services:
  sonarqube:
    image: sonarqube:lts-community
    container_name: sonarqube
    depends_on:
      - db
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    ports:
      - "9000:9000"

  db:
    image: postgres:15
    container_name: sonarqube-db
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
      POSTGRES_DB: sonar
    volumes:
      - postgresql:/var/lib/postgresql
      - postgresql_data:/var/lib/postgresql/data

volumes:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
  postgresql:
  postgresql_data:
```

Launch SonarQube:

```bash
sudo docker-compose up -d

# Verify containers are running
sudo docker-compose ps

# View logs
sudo docker-compose logs -f sonarqube
```

Access at `http://<EC2-PUBLIC-IP>:9000`.[^12]

***

## Part 3: Integration with Jenkins — Complete Demo

### Prerequisites

- Jenkins installed on the same or separate EC2 instance (port 8080)
- SonarQube running and accessible on port 9000
- A sample project (Java/Maven, Python, or Node.js) in a Git repository

### Step 1: Install SonarQube Plugin in Jenkins

```
Jenkins Dashboard → Manage Jenkins → Plugins → Available Plugins
→ Search "SonarQube Scanner for Jenkins" → Install → Restart Jenkins
```

This plugin provides:

- `withSonarQubeEnv` pipeline step
- `waitForQualityGate` pipeline step
- Global SonarQube server configuration
- SonarScanner tool auto-installation[^13][^9]

### Step 2: Generate Authentication Token in SonarQube

```
SonarQube Dashboard → Administration → Security → Users
→ Click on token icon for admin user
→ Generate Token → Name: "jenkins-token" → Type: Global Analysis Token
→ Copy the generated token (you won't see it again!)
```

Example token: `squ_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6`[^14]

### Step 3: Add SonarQube Token as Jenkins Credential

```
Jenkins Dashboard → Manage Jenkins → Credentials
→ System → Global credentials → Add Credentials
→ Kind: "Secret text"
→ Secret: <paste the SonarQube token>
→ ID: sonarqube-token
→ Description: SonarQube Authentication Token
→ Save
```

### Step 4: Configure SonarQube Server in Jenkins

```
Jenkins Dashboard → Manage Jenkins → System (Configure System)
→ Scroll to "SonarQube servers" section
→ Check "Environment variables"
→ Add SonarQube:
    Name: SonarQube-Server
    Server URL: http://<SONARQUBE-IP>:9000
    Server authentication token: sonarqube-token (the credential created above)
→ Save
```



### Step 5: Configure SonarScanner Tool in Jenkins

```
Jenkins Dashboard → Manage Jenkins → Tools (Global Tool Configuration)
→ Scroll to "SonarQube Scanner"
→ Add SonarQube Scanner:
    Name: SonarScanner
    Check "Install automatically"
    Version: SonarQube Scanner 5.x (latest)
→ Save
```



### Step 6: Configure Webhook in SonarQube (for Quality Gate)

For the `waitForQualityGate` step to work, SonarQube must call back to Jenkins:[^9]

```
SonarQube Dashboard → Administration → Configuration → Webhooks
→ Create
→ Name: Jenkins
→ URL: http://<JENKINS-IP>:8080/sonarqube-webhook/
→ Save
```

***

## Part 4: Demo Scenarios with Jenkinsfile Examples

### Scenario 1: Basic Java/Maven Project Analysis

This is the most common use case — scanning a Java project built with Maven:[^15]

**Jenkinsfile:**

```groovy
pipeline {
    agent any
    
    tools {
        maven 'Maven-3.9'
        jdk 'JDK-17'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/your-org/java-sample-app.git'
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Deploy') {
            steps {
                echo 'Deploying application...'
            }
        }
    }
    
    post {
        failure {
            echo 'Pipeline failed! Check SonarQube dashboard for details.'
        }
        success {
            echo 'Pipeline passed all quality checks!'
        }
    }
}
```

**What happens:**

1. Jenkins checks out the code from Git.
2. Maven compiles and runs unit tests.
3. `mvn sonar:sonar` runs the SonarQube analysis using the Maven plugin.
4. `waitForQualityGate` pauses the pipeline until SonarQube computes the quality gate result and sends it back via webhook.[^15][^9]
5. If the quality gate **passes**, the pipeline continues to deploy.
6. If the quality gate **fails**, the pipeline **aborts**.

### Scenario 2: Non-Maven Project Using SonarScanner CLI

For Python, Node.js, Go, or any non-Maven project, use the SonarScanner CLI:[^16][^17]

**Jenkinsfile:**

```groovy
pipeline {
    agent any
    
    environment {
        SCANNER_HOME = tool 'SonarScanner'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/your-org/python-flask-app.git'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=python-flask-app \
                        -Dsonar.projectName='Python Flask App' \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=./src \
                        -Dsonar.language=py \
                        -Dsonar.python.version=3.11 \
                        -Dsonar.sourceEncoding=UTF-8
                    """
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }
}
```

**Key Parameters Explained:**

| Parameter | Purpose |
|---|---|
| `sonar.projectKey` | Unique identifier for the project in SonarQube |
| `sonar.projectName` | Display name on SonarQube dashboard |
| `sonar.sources` | Path to the source code directory |
| `sonar.language` | Primary language (py, js, go, etc.) |
| `sonar.exclusions` | Files/folders to exclude (e.g., `**/test/**,**/node_modules/**`) |
| `sonar.tests` | Path to the test source directory |
| `sonar.python.coverage.reportPaths` | Path to coverage XML report |

### Scenario 3: Using sonar-project.properties File

Instead of passing parameters on the command line, create a `sonar-project.properties` file in the project root:

**sonar-project.properties:**

```properties
sonar.projectKey=my-nodejs-app
sonar.projectName=My Node.js App
sonar.projectVersion=2.0

sonar.sources=src
sonar.tests=test
sonar.exclusions=**/node_modules/**,**/dist/**,**/*.spec.js
sonar.test.inclusions=**/*.spec.js,**/*.test.js

sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.sourceEncoding=UTF-8
```

**Simplified Jenkinsfile:**

```groovy
pipeline {
    agent any
    
    environment {
        SCANNER_HOME = tool 'SonarScanner'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/your-org/nodejs-app.git'
            }
        }
        
        stage('Install & Test') {
            steps {
                sh 'npm install'
                sh 'npm test -- --coverage'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh "${SCANNER_HOME}/bin/sonar-scanner"
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }
}
```

The scanner automatically picks up `sonar-project.properties` from the workspace root.[^17]

### Scenario 4: Multi-Branch Pipeline with PR Analysis

For analyzing feature branches and pull requests (requires Developer Edition):

```groovy
pipeline {
    agent any
    
    tools {
        maven 'Maven-3.9'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build & Test') {
            steps {
                sh 'mvn clean verify'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh """
                        mvn sonar:sonar \
                        -Dsonar.projectKey=my-java-app \
                        -Dsonar.branch.name=${env.BRANCH_NAME}
                    """
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }
}
```

### Scenario 5: Full CI/CD Pipeline with SonarQube + Docker + Deploy

A real-world end-to-end pipeline:

```groovy
pipeline {
    agent any
    
    environment {
        SCANNER_HOME    = tool 'SonarScanner'
        DOCKER_IMAGE    = 'your-dockerhub/java-app'
        DOCKER_TAG      = "${BUILD_NUMBER}"
    }
    
    tools {
        maven 'Maven-3.9'
        jdk 'JDK-17'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/your-org/java-app.git',
                    credentialsId: 'github-creds'
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh """
                        mvn sonar:sonar \
                        -Dsonar.projectKey=java-app \
                        -Dsonar.projectName='Java Application' \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    """
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Docker Build & Push') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                        def app = docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                        app.push()
                        app.push('latest')
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    kubectl set image deployment/java-app \
                    java-app=${DOCKER_IMAGE}:${DOCKER_TAG} \
                    --namespace=production
                """
            }
        }
    }
    
    post {
        success {
            slackSend channel: '#deployments',
                      message: "✅ Build #${BUILD_NUMBER} passed and deployed!"
        }
        failure {
            slackSend channel: '#deployments',
                      message: "❌ Build #${BUILD_NUMBER} failed. Check SonarQube/Jenkins."
        }
    }
}
```

***

## Part 5: Working with SonarQube — Practical Commands

### Running SonarScanner Manually (Outside Jenkins)

If testing locally or from a standalone server:

```bash
# Install SonarScanner CLI
wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
unzip sonar-scanner-cli-5.0.1.3006-linux.zip
sudo mv sonar-scanner-5.0.1.3006-linux /opt/sonar-scanner
export PATH=$PATH:/opt/sonar-scanner/bin

# Run analysis
sonar-scanner \
  -Dsonar.host.url=http://<SONARQUBE-IP>:9000 \
  -Dsonar.login=squ_a1b2c3d4e5f6g7h8i9j0 \
  -Dsonar.projectKey=my-project \
  -Dsonar.sources=./src
```

### SonarQube API — Useful Commands

```bash
# Check server health
curl -u admin:password http://localhost:9000/api/system/health

# Get project quality gate status
curl -u admin:password \
  "http://localhost:9000/api/qualitygates/project_status?projectKey=my-project"

# Search for issues in a project
curl -u admin:password \
  "http://localhost:9000/api/issues/search?componentKeys=my-project&types=BUG&severities=CRITICAL"

# List all projects
curl -u admin:password \
  "http://localhost:9000/api/projects/search"

# Get project metrics (bugs, vulnerabilities, code_smells, coverage)
curl -u admin:password \
  "http://localhost:9000/api/measures/component?component=my-project&metricKeys=bugs,vulnerabilities,code_smells,coverage,duplicated_lines_density"
```

### Creating a Custom Quality Gate via UI

```
SonarQube → Quality Gates → Create
→ Name: "Strict Production Gate"
→ Add Conditions:
    - Coverage on New Code > 80%
    - Duplicated Lines on New Code < 3%
    - Reliability Rating on New Code = A (no bugs)
    - Security Rating on New Code = A (no vulnerabilities)
    - Maintainability Rating on New Code = A (no code smells)
→ Set as Default (optional)
```



### Creating a Custom Quality Profile

```
SonarQube → Quality Profiles → Create
→ Language: Java
→ Name: "Company Java Rules"
→ Parent: Sonar way (inherits all default rules)
→ Activate/Deactivate rules as needed
→ Assign to projects
```

***

## Part 6: Sample Code with Intentional Issues (For Demo)

### Java Sample with Bugs, Vulnerabilities, and Code Smells

Create a file `DemoApp.java` to demonstrate SonarQube detecting different issue types:

```java
import java.sql.*;
import java.util.*;

public class DemoApp {

    // BUG: Potential NullPointerException - using result without null check
    public String getUserName(Map<String, String> users, String id) {
        String name = users.get(id);
        return name.toUpperCase();  // Bug: name could be null
    }

    // VULNERABILITY: SQL Injection - user input concatenated directly
    public void getUser(String userId) throws Exception {
        Connection conn = DriverManager.getConnection("jdbc:mysql://localhost/db");
        Statement stmt = conn.createStatement();
        String query = "SELECT * FROM users WHERE id = '" + userId + "'";
        ResultSet rs = stmt.executeQuery(query);  // SQL Injection!
        // BUG: Connection and resources never closed (resource leak)
    }

    // VULNERABILITY: Hardcoded credentials
    private static final String DB_PASSWORD = "admin123";
    private static final String API_KEY = "sk-abc123secretkey";

    // CODE SMELL: Method too complex (high cyclomatic complexity)
    public String classifyScore(int score) {
        if (score > 90) {
            return "A+";
        } else if (score > 80) {
            return "A";
        } else if (score > 70) {
            return "B";
        } else if (score > 60) {
            return "C";
        } else if (score > 50) {
            return "D";
        } else if (score > 40) {
            return "E";
        } else {
            return "F";
        }
    }

    // CODE SMELL: Unused private method
    private void unusedMethod() {
        System.out.println("This method is never called");
    }

    // CODE SMELL: Empty catch block
    public void processData() {
        try {
            int result = 10 / 0;
        } catch (ArithmeticException e) {
            // Does nothing - swallows the exception
        }
    }

    // DUPLICATION: Copy-pasted logic
    public double calculateAreaCircle(double radius) {
        return 3.14159 * radius * radius;
    }

    public double calculateAreaCircle2(double r) {
        return 3.14159 * r * r;  // Duplicated logic
    }
}
```

When scanned, SonarQube will report:

- **2 Bugs**: Null pointer dereference, resource leak
- **3 Vulnerabilities**: SQL injection, 2 hardcoded credentials
- **3+ Code Smells**: Unused method, empty catch block, high complexity
- **1 Duplication**: The two calculateArea methods

### Python Sample with Issues

```python
import os
import subprocess

# VULNERABILITY: Command injection
def run_command(user_input):
    os.system("ls " + user_input)  # Command injection risk

# VULNERABILITY: Hardcoded secret
API_SECRET = "supersecret123"

# BUG: Division by zero risk
def calculate_average(numbers):
    total = sum(numbers)
    return total / len(numbers)  # Bug: crashes if list is empty

# CODE SMELL: Unused import (os imported but subprocess unused)
# CODE SMELL: Function too long (add 50+ lines to trigger)

# CODE SMELL: Duplicated string literal
def get_status():
    print("STATUS: OK")
    print("STATUS: OK")
    print("STATUS: OK")
```

***

## Part 7: Troubleshooting Common Issues

| Problem | Cause | Solution |
|---|---|---|
| SonarQube not starting | Insufficient memory or `vm.max_map_count` too low | Ensure `vm.max_map_count=524288`, use t2.medium+ instance |
| Cannot connect to SonarQube from browser | Port 9000 not open in Security Group | Add inbound rule for TCP 9000 in EC2 Security Group |
| `waitForQualityGate` hangs forever | Webhook not configured or SonarQube can't reach Jenkins | Verify webhook URL in SonarQube → Administration → Webhooks |
| "Not authorized" error in Jenkins | Token expired or wrong credential type | Regenerate token, ensure credential type is "Secret text" |
| SonarScanner not found | Tool not configured in Jenkins Global Tool Configuration | Add SonarScanner in Manage Jenkins → Tools |
| PostgreSQL connection refused | PostgreSQL not running or wrong credentials | Check `systemctl status postgresql`, verify `sonar.properties` |
| Analysis succeeds but no results in dashboard | Wrong `sonar.projectKey` or server URL | Verify projectKey matches and server URL is correct |

***

## Part 8: Architecture & Flow Diagram (Text)

```
Developer → Git Push → Jenkins (CI/CD)
                           │
                    ┌──────┴──────┐
                    │  Build &    │
                    │  Unit Test  │
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │ SonarScanner│ ← Scans code, sends to SonarQube
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │  SonarQube  │ ← Analyzes, computes Quality Gate
                    │   Server    │
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │  Webhook    │ ← Sends pass/fail to Jenkins
                    └──────┬──────┘
                           │
                   Pass?──┤
                  /        \
                YES        NO
                 │          │
              Deploy    Pipeline
              to Prod   Fails ❌
```

The flow works as follows:[^13][^9]

1. A developer pushes code to Git.
2. Jenkins triggers the pipeline automatically.
3. The code is built and unit tests run.
4. SonarScanner analyzes the code and pushes results to SonarQube Server.
5. SonarQube processes the results, runs rules from the Quality Profile, and evaluates the Quality Gate.
6. SonarQube sends the pass/fail result back to Jenkins via webhook.
7. If the Quality Gate passes, the pipeline continues to deployment. If it fails, the pipeline aborts.

***

## Quick Reference: Essential Commands

```bash
# ---- SonarQube Service Management ----
sudo systemctl start sonarqube
sudo systemctl stop sonarqube
sudo systemctl restart sonarqube
sudo systemctl status sonarqube
journalctl -u sonarqube -f

# ---- SonarQube Logs ----
tail -f /opt/sonarqube/logs/sonar.log
tail -f /opt/sonarqube/logs/web.log
tail -f /opt/sonarqube/logs/es.log
tail -f /opt/sonarqube/logs/ce.log

# ---- Docker Compose (if using Docker) ----
sudo docker-compose up -d
sudo docker-compose down
sudo docker-compose logs -f sonarqube
sudo docker-compose ps

# ---- PostgreSQL ----
sudo systemctl status postgresql
sudo -u postgres psql -c "SELECT datname FROM pg_database;"
```

---

## References

1. [What is SonarQube and use cases of SonarQube? - DevOpsSchool ...](https://www.devopsschool.com/blog/what-is-sonarqube-and-use-cases-of-sonarqube/) - It provides a range of static code analysis and code review features to help development teams ident...

2. [What is SonarQube?](https://www.geeksforgeeks.org/devops/sonarqube/) - Your All-in-One Learning Portal: GeeksforGeeks is a comprehensive educational platform that empowers...

3. [How To Install SonarQube on Ubuntu 22.04](https://hostnextra.com/learn/tutorials/how-to-install-sonarqube-on-ubuntu)

4. [What is a Quality Gate in SonarQube?](https://www.bitegarden.com/what-is-quality-gate-sonarqube) - We teach you to use the Quality Gates to guarantee the security and quality of your code

5. [Types of Scanning / Issues Detected by SonarQube - LinkedIn](https://www.linkedin.com/pulse/types-scanning-issues-detected-sonarqube-vineeth-kumar-q6hlc) - When SonarQube runs a scan, it performs static code analysis to detect issues in several categories:...

6. [Issues](https://docs.sonarsource.com/sonarqube-server/10.4/user-guide/issues/) - While running an analysis, SonarQube raises an issue every time a piece of code breaks a coding rule...

7. [Some SonarQube issues have a significant but small effect on faults ...](https://www.sciencedirect.com/science/article/abs/pii/S0164121220301734) - SonarQube classifies issues into three main categories: Code Smells, i.e., issues that increase chan...

8. [Quality gates](https://sonarqube-documentation.netlify.app/latest/user-guide/quality-gates/) - Quality Gates enforce a quality policy in your organization by answering one question: is my project...

9. [Key features | Sonar Documentation](https://docs.sonarsource.com/sonarqube-server/10.8/analyzing-source-code/ci-integration/jenkins-integration/key-features/) - Sonar provides an extension for Jenkins to enable smooth integration with Jenkins. This section expl...

10. [How to Install SonarQube 9.9.2 LTS on an EC2 Instance Running ...](https://mantratech.hashnode.dev/how-to-install-sonarqube-on-ubuntu) - In this step-by-step guide, we will walk you through the process of setting up SonarQube 9.9.2 LTS o...

11. [How to set up SonarQube on AWS Ubuntu EC2 - Cloudkul](https://cloudkul.com/blog/how-to-set-up-sonarqube-on-aws-ubuntu-ec2/) - Run Ubuntu system update. # apt update SonarQube In my case, java is already installed if not instal...

12. [GitHub - amscotti/SonarQubeCompose: SonarQube with Docker Compose](https://github.com/amscotti/SonarQubeCompose) - SonarQube with Docker Compose. Contribute to amscotti/SonarQubeCompose development by creating an ac...

13. [SonarQube Server integration with Jenkins | Documentation](https://docs.sonarsource.com/sonarqube-server/latest/analyzing-source-code/ci-integration/jenkins-integration/key-features/) - SonarSource provides an extension for Jenkins to enable smooth integration with Jenkins. This sectio...

14. [How To Integrate SonarQube With Jenkins? - GeeksforGeeks](https://www.geeksforgeeks.org/devops/how-to-integrate-sonarqube-with-jenkins/) - Your All-in-One Learning Portal: GeeksforGeeks is a comprehensive educational platform that empowers...

15. [SonarQube Scanner for Jenkins](https://www.jenkins.io/doc/pipeline/steps/sonar/) - Jenkins – an open source automation server which enables developers around the world to reliably bui...

16. [How to execute sonarqube scanner using jenkins pipeline?](https://www.devopsschool.com/blog/how-to-execute-sonarqube-scanner-using-jenkins-pipeline/) - The tool name "SonarQube Scanner 2.8" needs to match the "Name" field of a SonarQube Scanner Install...

17. [SonarQube Scans in Jenkins Declarative Pipeline](https://igorski.co/sonarqube-scans-using-jenkins-declarative-pipelines/) - Setup a SonarQube scan using Jenkins declarative pipelines. Find out how to use the withSonarQubeEnv...

