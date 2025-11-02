# Jenkins Build Artifact Deployment to AWS QA Server

## Overview
This guide covers various approaches for deploying locally generated Jenkins build artifacts (.war/.jar files) to a QA server hosted in AWS Cloud.

---

## Approach 1: SSH/SCP Direct Transfer

### Description
Direct file transfer from Jenkins to EC2 instance using SSH/SCP protocols.

### Flow Diagram
```
┌─────────────┐
│   Jenkins   │
│   (Local)   │
└──────┬──────┘
       │
       │ 1. Build artifact (.war/.jar)
       │
       ├──────────────────────────────┐
       │                              │
       │ 2. SCP/SFTP transfer         │
       │    via SSH                   │
       │                              │
       ▼                              │
┌─────────────────┐                  │
│  AWS EC2 (QA)   │                  │
│                 │                  │
│  - Receives     │◄─────────────────┘
│    artifact     │
│  - Deploys to   │
│    app server   │
└─────────────────┘
```

### Implementation Steps
1. **Configure SSH credentials in Jenkins**
   - Install "Publish Over SSH" plugin
   - Add QA server SSH key in Jenkins credentials

2. **Jenkins Pipeline Example**
   ```groovy
   stage('Deploy to QA') {
       steps {
           sshPublisher(
               publishers: [
                   sshPublisherDesc(
                       configName: 'QA-Server',
                       transfers: [
                           sshTransfer(
                               sourceFiles: 'target/*.war',
                               removePrefix: 'target',
                               remoteDirectory: '/opt/tomcat/webapps'
                           )
                       ]
                   )
               ]
           )
       }
   }
   ```

3. **QA Server Setup**
   - Open port 22 in Security Group
   - Configure SSH key authentication
   - Set proper file permissions

### Pros & Cons
✅ **Pros:**
- Simple and straightforward
- No intermediate storage needed
- Direct control over deployment

❌ **Cons:**
- Requires opening SSH port to Jenkins IP
- No built-in versioning or rollback
- Single point of failure

---

## Approach 2: S3 as Intermediary Storage

### Description
Jenkins uploads artifacts to S3, then QA server pulls from S3.

### Flow Diagram
```
┌─────────────┐
│   Jenkins   │
│   (Local)   │
└──────┬──────┘
       │
       │ 1. Build artifact
       │
       │ 2. Upload to S3
       │    (AWS CLI/Plugin)
       │
       ▼
┌─────────────────┐
│   Amazon S3     │
│   Bucket        │
│                 │
│ /artifacts/     │
│   app-v1.war    │
└────────┬────────┘
         │
         │ 3. Download artifact
         │    (AWS CLI/SDK)
         │
         ▼
┌─────────────────┐
│  AWS EC2 (QA)   │
│                 │
│  - Pulls from   │
│    S3           │
│  - Deploys      │
│    locally      │
└─────────────────┘
```

### Implementation Steps

1. **Jenkins Configuration**
   ```groovy
   stage('Upload to S3') {
       steps {
           withAWS(credentials: 'aws-credentials', region: 'us-east-1') {
               s3Upload(
                   bucket: 'my-jenkins-artifacts',
                   path: "builds/${env.BUILD_NUMBER}/",
                   includePathPattern: '**/*.war'
               )
           }
       }
   }
   ```

2. **QA Server Deployment Script**
   ```bash
   #!/bin/bash
   # download-and-deploy.sh
   
   BUILD_NUMBER=$1
   S3_BUCKET="my-jenkins-artifacts"
   APP_NAME="myapp"
   
   # Download from S3
   aws s3 cp s3://${S3_BUCKET}/builds/${BUILD_NUMBER}/${APP_NAME}.war \
             /tmp/${APP_NAME}.war
   
   # Stop application
   systemctl stop tomcat
   
   # Deploy
   cp /tmp/${APP_NAME}.war /opt/tomcat/webapps/
   
   # Start application
   systemctl start tomcat
   ```

3. **IAM Setup**
   - Jenkins: IAM user with S3 PutObject permissions
   - QA Server: IAM role with S3 GetObject permissions

### Pros & Cons
✅ **Pros:**
- Decouples build from deployment
- Artifact versioning via S3
- Can enable S3 versioning for rollbacks
- No direct network connection needed

❌ **Cons:**
- Manual deployment step on QA server
- Additional S3 costs
- Need to manage S3 lifecycle policies

---

## Approach 3: AWS CodeDeploy

### Description
Enterprise-grade deployment using AWS CodeDeploy service with full automation and rollback capabilities.

### Flow Diagram
```
┌─────────────┐
│   Jenkins   │
│   (Local)   │
└──────┬──────┘
       │
       │ 1. Build artifact
       │
       │ 2. Upload to S3
       │    with appspec.yml
       │
       ▼
┌─────────────────┐
│   Amazon S3     │
│   Bucket        │
└────────┬────────┘
         │
         │ 3. Trigger deployment
         │
         ▼
┌─────────────────┐
│  AWS CodeDeploy │
│                 │
│  - Deployment   │
│    Group        │
│  - Manages      │
│    lifecycle    │
└────────┬────────┘
         │
         │ 4. Deploy via agent
         │    (Install/Start/Validate)
         │
         ▼
┌─────────────────┐
│  AWS EC2 (QA)   │
│                 │
│  - CodeDeploy   │
│    Agent        │
│  - Auto deploy  │
│  - Health check │
└─────────────────┘
```

### Implementation Steps

1. **Create appspec.yml**
   ```yaml
   version: 0.0
   os: linux
   files:
     - source: /myapp.war
       destination: /opt/tomcat/webapps
   hooks:
     BeforeInstall:
       - location: scripts/stop_server.sh
         timeout: 300
     AfterInstall:
       - location: scripts/start_server.sh
         timeout: 300
     ValidateService:
       - location: scripts/validate.sh
         timeout: 300
   ```

2. **Jenkins Pipeline**
   ```groovy
   stage('Deploy to QA') {
       steps {
           // Upload to S3
           withAWS(credentials: 'aws-creds', region: 'us-east-1') {
               s3Upload(
                   bucket: 'codedeploy-artifacts',
                   path: "myapp/${env.BUILD_NUMBER}/",
                   includePathPattern: '**/*'
               )
               
               // Trigger CodeDeploy
               step([
                   $class: 'AWSCodeDeployPublisher',
                   applicationName: 'MyApp',
                   deploymentGroupName: 'QA-Group',
                   deploymentConfig: 'CodeDeployDefault.OneAtATime',
                   s3bucket: 'codedeploy-artifacts',
                   s3prefix: "myapp/${env.BUILD_NUMBER}/",
                   region: 'us-east-1'
               ])
           }
       }
   }
   ```

3. **QA Server Setup**
   - Install CodeDeploy agent
   - Attach IAM role with CodeDeploy permissions
   - Tag EC2 instance for deployment group

### Pros & Cons
✅ **Pros:**
- Built-in rollback capability
- Deployment lifecycle hooks
- Health monitoring
- Supports blue/green deployments
- Multi-instance deployments

❌ **Cons:**
- More complex setup
- Additional AWS service costs
- Learning curve for appspec.yml

---

## Approach 4: Container Registry (ECR)

### Description
Containerize the application and deploy via Docker containers using AWS ECR.

### Flow Diagram
```
┌─────────────┐
│   Jenkins   │
│   (Local)   │
└──────┬──────┘
       │
       │ 1. Build artifact
       │
       │ 2. Build Docker image
       │    FROM tomcat:9
       │    COPY app.war /webapps/
       │
       │ 3. Push to ECR
       │    docker push ecr.../myapp:tag
       │
       ▼
┌─────────────────┐
│   Amazon ECR    │
│  (Container     │
│   Registry)     │
└────────┬────────┘
         │
         │ 4. Pull image
         │    docker pull
         │
         ▼
┌─────────────────┐
│  AWS EC2/ECS    │
│  (QA)           │
│                 │
│  - Pull image   │
│  - Run container│
│  - Port mapping │
└─────────────────┘
```

### Implementation Steps

1. **Dockerfile**
   ```dockerfile
   FROM tomcat:9-jdk11
   
   # Remove default webapps
   RUN rm -rf /usr/local/tomcat/webapps/*
   
   # Copy artifact
   COPY target/myapp.war /usr/local/tomcat/webapps/ROOT.war
   
   EXPOSE 8080
   
   CMD ["catalina.sh", "run"]
   ```

2. **Jenkins Pipeline**
   ```groovy
   stage('Build and Push Docker Image') {
       steps {
           script {
               def app = docker.build("myapp:${env.BUILD_NUMBER}")
               
               docker.withRegistry(
                   'https://123456789.dkr.ecr.us-east-1.amazonaws.com',
                   'ecr:us-east-1:aws-credentials'
               ) {
                   app.push("${env.BUILD_NUMBER}")
                   app.push("latest")
               }
           }
       }
   }
   
   stage('Deploy to QA') {
       steps {
           sshPublisher(
               publishers: [
                   sshPublisherDesc(
                       configName: 'QA-Server',
                       transfers: [
                           sshTransfer(
                               execCommand: """
                                   aws ecr get-login-password --region us-east-1 | \
                                   docker login --username AWS --password-stdin \
                                   123456789.dkr.ecr.us-east-1.amazonaws.com
                                   
                                   docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:${env.BUILD_NUMBER}
                                   
                                   docker stop myapp-qa || true
                                   docker rm myapp-qa || true
                                   
                                   docker run -d --name myapp-qa \
                                     -p 8080:8080 \
                                     123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:${env.BUILD_NUMBER}
                               """
                           )
                       ]
                   )
               ]
           )
       }
   }
   ```

3. **QA Server Setup**
   ```bash
   # Install Docker
   yum install -y docker
   systemctl start docker
   systemctl enable docker
   
   # Install AWS CLI
   curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
   unzip awscliv2.zip
   sudo ./aws/install
   ```

### Pros & Cons
✅ **Pros:**
- Consistent deployment across environments
- Easy rollback (switch image tags)
- Isolation and portability
- Can scale to ECS/EKS easily

❌ **Cons:**
- Requires Docker knowledge
- Additional image storage costs
- Overhead for simple apps
- Need to manage container lifecycle

---

## Approach 5: Configuration Management (Ansible)

### Description
Use Ansible playbooks triggered by Jenkins to orchestrate deployment.

### Flow Diagram
```
┌─────────────┐
│   Jenkins   │
│   (Local)   │
└──────┬──────┘
       │
       │ 1. Build artifact
       │
       │ 2. Upload to S3
       │
       │ 3. Trigger Ansible
       │    playbook
       │
       ▼
┌─────────────────┐
│   Ansible       │
│   Controller    │
│   (Jenkins or   │
│    separate)    │
└────────┬────────┘
         │
         │ 4. Execute playbook
         │    - Download from S3
         │    - Stop service
         │    - Deploy artifact
         │    - Start service
         │    - Verify
         │
         ▼
┌─────────────────┐
│  AWS EC2 (QA)   │
│                 │
│  - Receives     │
│    commands     │
│  - Executes     │
│    tasks        │
└─────────────────┘
```

### Implementation Steps

1. **Ansible Playbook (deploy-app.yml)**
   ```yaml
   ---
   - name: Deploy application to QA
     hosts: qa_servers
     become: yes
     vars:
       app_name: myapp
       build_number: "{{ lookup('env', 'BUILD_NUMBER') }}"
       s3_bucket: jenkins-artifacts
       tomcat_webapps: /opt/tomcat/webapps
     
     tasks:
       - name: Download artifact from S3
         aws_s3:
           bucket: "{{ s3_bucket }}"
           object: "builds/{{ build_number }}/{{ app_name }}.war"
           dest: "/tmp/{{ app_name }}.war"
           mode: get
       
       - name: Stop Tomcat
         systemd:
           name: tomcat
           state: stopped
       
       - name: Backup existing application
         copy:
           src: "{{ tomcat_webapps }}/{{ app_name }}.war"
           dest: "{{ tomcat_webapps }}/{{ app_name }}.war.backup"
           remote_src: yes
         ignore_errors: yes
       
       - name: Deploy new artifact
         copy:
           src: "/tmp/{{ app_name }}.war"
           dest: "{{ tomcat_webapps }}/{{ app_name }}.war"
           owner: tomcat
           group: tomcat
           mode: '0644'
           remote_src: yes
       
       - name: Start Tomcat
         systemd:
           name: tomcat
           state: started
       
       - name: Wait for application to start
         uri:
           url: "http://localhost:8080/{{ app_name }}/health"
           status_code: 200
         register: result
         until: result.status == 200
         retries: 10
         delay: 5
   ```

2. **Ansible Inventory (inventory.ini)**
   ```ini
   [qa_servers]
   qa-server-1 ansible_host=10.0.1.50 ansible_user=ec2-user
   qa-server-2 ansible_host=10.0.1.51 ansible_user=ec2-user
   
   [qa_servers:vars]
   ansible_ssh_private_key_file=~/.ssh/qa-key.pem
   ansible_python_interpreter=/usr/bin/python3
   ```

3. **Jenkins Pipeline**
   ```groovy
   stage('Deploy with Ansible') {
       steps {
           // Upload artifact to S3
           withAWS(credentials: 'aws-credentials', region: 'us-east-1') {
               s3Upload(
                   bucket: 'jenkins-artifacts',
                   path: "builds/${env.BUILD_NUMBER}/",
                   includePathPattern: '**/*.war'
               )
           }
           
           // Run Ansible playbook
           ansiblePlaybook(
               playbook: 'deploy-app.yml',
               inventory: 'inventory.ini',
               credentialsId: 'ansible-ssh-key',
               extraVars: [
                   BUILD_NUMBER: env.BUILD_NUMBER
               ]
           )
       }
   }
   ```

### Pros & Cons
✅ **Pros:**
- Declarative and idempotent
- Can manage full server configuration
- Multi-server deployments
- Reusable playbooks
- Good for complex deployments

❌ **Cons:**
- Requires Ansible knowledge
- Additional tooling to maintain
- SSH access needed
- More complex than direct methods

---

## Comparison Matrix

| Approach | Complexity | Rollback | Multi-Instance | Cost | Best For |
|----------|-----------|----------|----------------|------|----------|
| **SSH/SCP** | Low | Manual | Difficult | Low | Simple setups, single server |
| **S3 Intermediary** | Low-Medium | Manual | Manual | Low | Decoupled build/deploy |
| **CodeDeploy** | Medium-High | Automated | Built-in | Medium | Enterprise, compliance |
| **ECR/Containers** | Medium | Easy | Easy | Medium | Microservices, scalability |
| **Ansible** | Medium | Scripted | Built-in | Low | Multi-server, config mgmt |

---

## Recommended Approach for AWS

**For most AWS deployments: S3 + CodeDeploy**

This combination provides:
- ✅ AWS-native integration
- ✅ Automated rollback capabilities
- ✅ Deployment lifecycle management
- ✅ Scalable to multiple instances
- ✅ Built-in health checks
- ✅ Compliance and audit trails

### Quick Start Implementation

1. **Set up S3 bucket** for artifacts
2. **Install CodeDeploy agent** on QA server
3. **Create CodeDeploy application** and deployment group
4. **Configure Jenkins** with AWS credentials
5. **Create appspec.yml** with deployment hooks
6. **Add deployment stage** to Jenkins pipeline

---

## Security Considerations

### All Approaches Should Include:
- ✅ Use IAM roles (not access keys) on EC2 instances
- ✅ Encrypt artifacts in S3 with KMS
- ✅ Use VPC endpoints for S3 access (avoid internet)
- ✅ Implement least-privilege IAM policies
- ✅ Enable CloudTrail for audit logging
- ✅ Use SSL/TLS for all transfers
- ✅ Scan artifacts for vulnerabilities before deployment
- ✅ Use AWS Secrets Manager for credentials

---

## Troubleshooting Tips

### Common Issues:

1. **Deployment fails silently**
   - Check IAM permissions
   - Review CloudWatch logs
   - Verify Security Group rules

2. **Artifact not found**
   - Confirm S3 bucket path
   - Check S3 bucket policy
   - Verify IAM role permissions

3. **Service doesn't start after deployment**
   - Check application logs
   - Verify file permissions
   - Ensure dependencies are met

4. **Rollback needed**
   - CodeDeploy: Use built-in rollback
   - S3: Deploy previous version
   - Containers: Switch to previous tag

---

## Additional Resources

- [AWS CodeDeploy Documentation](https://docs.aws.amazon.com/codedeploy/)
- [Jenkins AWS Plugins](https://plugins.jenkins.io/aws-credentials/)
- [AWS CLI Reference](https://docs.aws.amazon.com/cli/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Ansible AWS Guide](https://docs.ansible.com/ansible/latest/collections/amazon/aws/)

---

*Last Updated: 2025-11-02*
