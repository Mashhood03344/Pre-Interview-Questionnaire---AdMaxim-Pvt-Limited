# Pre-Interview Questionnaire - AdMaxim Pvt Limited


# Table of Contents

  - [Question 1](#question-1)
  - [Optimized Dockerfile for Building and Deploying Java Application](#optimized-dockerfile-for-building-and-deploying-java-application)
  - [Commands to Build and Run the Docker Image](#commands-to-build-and-run-the-docker-image)
  - [Explanation of the Optimization in the Dockerfile](#explanation-of-the-optimization-in-the-dockerfile)
  - [Question 2](#question-2)
  - [Deployment.yaml and Service.yaml Files](#deploymentyaml-and-serviceyaml-files)
    - [Deployment.yaml](#deploymentyaml)
    - [service.yaml](#serviceyaml)
  - [Command to apply the deployment and the service](#command-to-apply-the-deployment-and-the-service)
  - [Command to see all the pods in the cluster](#command-to-see-all-the-pods-in-the-cluster)
  - [Command to see all the services in the cluster](#command-to-see-all-the-services-in-the-cluster)
  - [Command to see the IP address of the Load Balancer](#command-to-see-the-ip-address-of-the-load-balancer)
  - [Command to delete Deployment and service](#command-to-delete-deployment-and-service)
  - [Question 3](#question-3)
  - [Bastion Host](#bastion-host)
  - [Role of the Bastion Host in a VPC Setup](#role-of-the-bastion-host-in-a-vpc-setup)
  - [How the Bastion Host Works in This Setup](#how-the-bastion-host-works-in-this-setup)
  - [Example Use Case](#example-use-case)
  - [Question 4](#question-4)
  - [process_logs.sh](#process-logssh)
  - [Commands to Run the Script](#commands-to-run-the-script)
  - [Email Notification Configuration](#email-notification-configuration)
  - [Question 5](#question-5)
  - [What is Amazon S3?](#what-is-amazon-s3)
  - [Key Features of Amazon S3](#key-features-of-amazon-s3)
  - [Securing Sensitive Data Stored in Amazon S3](#securing-sensitive-data-stored-in-amazon-s3)
  - [Optimizing Cost for Storing Large Amounts of Data](#optimizing-cost-for-storing-large-amounts-of-data)


## Question 1

**Create a optimized docker file to Build and Deploy a Java Application. The source code is hosted on a GitHub repository (assumed):
https://github.com/admaxim-user/java-project.git                      
(After build the war file locate in java-project/target/app-java.war). The final application should be deployed as ROOT.war in a Tomcat server and expose port 8080. Give docker commands to build your docker file.**

Optimized Dockerfile for Building and Deploying Java Application
              
		      
	# Stage 1: Build the application using Maven (Alpine version)
	FROM maven:3.8-openjdk-11-slim AS builder

	# Install Git in the Maven image
	RUN apt-get update && apt-get install -y git && apt-get clean

	# Set the working directory to /app
	WORKDIR /app

	# Clone the repository if not already cloned and pull the latest changes
	RUN if [ ! -d "/app/java-calculator-app" ]; then \
		git clone https://github.com/Mashhood03344/java-calculator-app.git; \
	    else \
		cd java-calculator-app && git pull; \
	    fi

	# Navigate to the cloned repository
	WORKDIR /app/java-calculator-app

	# Build the WAR file (skip tests for faster build)
	RUN mvn clean package -DskipTests && \
	    # Clean up the Maven cache to reduce the size
	    rm -rf ~/.m2/repository

	# Stage 2: Deploy the application with Tomcat (Alpine version)
	FROM tomcat:9-alpine

	# Set the working directory to Tomcat's webapps folder
	WORKDIR /usr/local/tomcat/webapps

	# Copy the WAR file from the build stage to Tomcat as ROOT.war
	COPY --from=builder /app/java-calculator-app/target/app-java-1.0.0.war /usr/local/tomcat/webapps/ROOT.war

	# Expose port 8080 to the outside world (Tomcat default)
	EXPOSE 8080

	# Start Tomcat server
	CMD ["catalina.sh", "run"]

	# REPOSITORY        TAG       IMAGE ID       CREATED          SIZE
	# java-tomcat-app   latest    29c6d5c0145b   55 seconds ago   444MB


	# END RESULT 

	# REPOSITORY        TAG       IMAGE ID       CREATED          SIZE
	# java-tomcat-app   latest    a615c8eb7bf8   46 seconds ago   117MB


### Commands to Build and Run the Docker Image

**Build the Docker Image**

Run the following command in the directory containing the Dockerfile:

	docker build -t java-tomcat-app:latest .

***Run the Docker Container**

Start the container with the following command:

	docker run -d -p 3000:3000 --name java-tomcat-container java-tomcat-app:latest
	
**Verify the Running Container**

Check if the container is running:

	docker ps

**Access the Application**

Open a browser or use curl to access the deployed application:

	curl http://localhost:8080

### Explanation of the Optimization in the Dockerfile

The optimizations to reduce the size of the Docker image in the Dockerfile can be broken down as follows:

1. **Multi-Stage Build:**

    **Stage 1: Build Stage (Maven + Git):**
    
        The first stage uses the maven:3.8-openjdk-11-slim image for building the application. This stage includes all the tools needed for the build process (Maven, Git, etc.), but these tools are not needed in the final runtime image. By separating the build process from the runtime environment, unnecessary tools do not end up in the final image.
    
    **Stage 2: Runtime Stage (Tomcat):**
    
        The second stage uses the tomcat:9-alpine image, which is a much smaller, lightweight version of Tomcat (based on Alpine Linux). This image includes only the Tomcat server and necessary components for running the application. The build tools like Maven and Git are excluded from this stage, keeping the final image smaller.

2. **Cleaning Up After Git Installation:**

    After installing Git in the build stage, the following command is used to clean up unnecessary files:

	```bash
	 apt-get clean
	 ```   

 This removes any temporary files created during the installation process, further reducing the image size.

3. **Removing the Maven Cache:**

    After building the project with Maven, the Dockerfile removes the Maven local repository to reduce the final image size:

	```bash
	rm -rf ~/.m2/repository
	```

This step is important because Maven caches downloaded dependencies in the local repository. While this helps speed up future builds, it is not required for running the application. Removing it from the final image reduces unnecessary bloat.

4. **Optimized Copying:**

    The Dockerfile uses the COPY command efficiently:

	```bash
	COPY --from=builder /app/target/app-java-1.0.0.war /usr/local/tomcat/webapps/ROOT.war
	```

This ensures that only the necessary WAR file is copied into the final image, and no other build-related files (such as source code or build dependencies) are included in the final runtime image.

## Question 2

**Using your custom Docker image (refer to the previous question), create a Kubernetes deployment file to deploy the application. The deployment should use a ReplicaSet of 3 and expose the application via a LoadBalancer service.**


### Deployment.yaml and Service.yaml Files


 **deployment.yaml**

	```bash
	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: java-tomcat-app-deployment
	spec:
	  replicas: 3
	  selector:
	    matchLabels:
	      app: java-tomcat-app
	  template:
	    metadata:
	      labels:
		app: java-tomcat-app
	    spec:
	      containers:
		- name: java-tomcat-app
		  image: mashhood03344/java-tomcat-app:latest   # Docker image from Docker Hub
		  ports:
		    - containerPort: 8080   # Expose port 8080 from the container
	---
	apiVersion: apps/v1
	kind: ReplicaSet
	metadata:
	  name: java-tomcat-app-replicaset
	spec:
	  replicas: 3
	  selector:
	    matchLabels:
	      app: java-tomcat-app
	  template:
	    metadata:
	      labels:
		app: java-tomcat-app
	    spec:
	      containers:
		- name: java-tomcat-app
		  image: mashhood03344/java-tomcat-app:latest   # Docker image from Docker Hub
		  ports:
		    - containerPort: 8080
	```

**service.yaml**

	```bash
	apiVersion: v1
	kind: Service
	metadata:
	  name: java-tomcat-app-service
	spec:
	  selector:
	    app: java-tomcat-app  # Matches the app label from the deployment
	  ports:
	    - protocol: TCP
	      port: 80      # Exposed port on the LoadBalancer
	      targetPort: 8080   # The port on the container (Tomcat default)
	  type: LoadBalancer   # Exposes the service externally
	```

### Command apply the deployment and the service

	```bash
	kubectl apply -f k8s/deployment.yaml
	kubectl apply -f k8s/service.yaml
	```

### Command to see all the pods in the cluster

	```bash
	kubectl get pods
	```
	
### Command to see all the services in the cluster

	```bash
	kubectl get services
	```
	
### Command to see the IP address of the Load Balancer 

 Once the LoadBalancer is set up, you should be able to access your application via the external IP address provided by the LoadBalancer.
You can get the external IP using:

	```bash
	kubectl get svc java-tomcat-app-service
	```

Look under the EXTERNAL-IP column to find the IP address of the LoadBalancer.

### Command to delete Deployment and service

**To delete Deployment**

	```bash
	kubectl delete deployment deployment.yaml
	```
	
**To delete Service**

	```bash
	kubectl delete service service.yaml
	```

## Question 3 

**You are tasked with setting up a Virtual Private Cloud (VPC) with both public and private subnets for deploy ec2 instance in both subnets.  Explain what a Bastion Host is and its role in this setup.**

### Bastion Host

A Bastion Host (sometimes called a Jump Host) is a special-purpose server used to securely access resources in a private network or subnet that cannot be accessed directly from the internet. It acts as a bridge or intermediary between the external world (public internet) and the internal network (private subnets) by allowing secure access to instances in the private subnets. In a typical setup with public and private subnets, the Bastion Host serves as the access point for administrators or users who need to manage or interact with instances in the private subnet.
Role of the Bastion Host in a VPC Setup
In a Virtual Private Cloud (VPC) with both public and private subnets, the Bastion Host is deployed in the public subnet to facilitate secure SSH or RDP access to EC2 instances in the private subnet.
Here’s how the Bastion Host fits into the setup:

1. **Access to Private Instances:**

- EC2 instances in private subnets cannot have direct access from the internet for security reasons. They are isolated from the public internet.
- The Bastion Host in the public subnet provides a secure entry point. Admins or authorized users can SSH into the Bastion Host (which is publicly accessible) and from there, they can SSH into instances in the private subnet.

2. **Security Considerations:**

- Access Control: The Bastion Host is typically hardened and tightly controlled. Access is granted based on SSH keys or other secure methods, ensuring that only authorized users can access it.
- Minimal Attack Surface: The Bastion Host is the only instance that has inbound internet traffic (usually limited to SSH or RDP), which minimizes the attack surface of the private network.
- Audit and Monitoring: The Bastion Host can be used for logging and monitoring all access to private instances, which helps in auditing and detecting any suspicious activities.
- Networking Setup:
- The public subnet contains the Bastion Host, which is assigned a public IP to be accessible from the internet.
- The private subnet contains the EC2 instances that need to be accessed securely. These instances do not have public IP addresses and can only communicate with the Bastion Host via private IP addresses.
- The Bastion Host must be configured with appropriate security groups to allow SSH or RDP access from trusted IP addresses (typically your office or VPN).

## How the Bastion Host Works in This Setup

1. **SSH or RDP Connection to Bastion Host:**

-First, the user connects to the Bastion Host over the internet using SSH (for Linux) or RDP (for Windows). The Bastion Host should only allow inbound connections on ports such as 22 (SSH) or 3389 (RDP) from trusted IPs.

2.**SSH Connection from Bastion Host to Private Instances:**

- After logging into the Bastion Host, the user can SSH into the EC2 instances in the private subnet using the private IP of the instances.
- The private instances don’t have direct public internet access, but by hopping through the Bastion Host, users can securely manage those instances.

3. **Security Group Setup:**

- The Bastion Host has a security group that allows inbound access on the required ports (e.g., port 22 for SSH or port 3389 for RDP) only from trusted IP addresses (such as the user's home office IP, VPN, or corporate network).
- The EC2 instances in the private subnet have security groups that only allow inbound connections from the Bastion Host’s private IP address, preventing direct access from the public internet.

## Example Use Case

1. Public Subnet: Contains the Bastion Host (EC2 instance with a public IP).
2. Private Subnet: Contains EC2 instances that only have private IPs and cannot be accessed directly from the internet.
3. **Access Flow:**
 - User connects to the Bastion Host via SSH (using the public IP of the Bastion Host).
 - From the Bastion Host, the user can SSH to the private instances using their private IPs.

This setup ensures that sensitive infrastructure in private subnets is protected, while still allowing necessary administrative access via the Bastion Host.

## Question 4

**Write a shell script that processes a log file (/var/log/app.log) and outputs a summary of the number of occurrences of each error type (e.g., "ERROR", "WARN", "INFO") in the log. Include error handling & Send an email notification if any of the error types exceeds a count of 10 in this script.**

## process_logs.sh

	```bash
	#!/bin/bash

	# Log file location
	LOG_FILE="/var/log/app.log"

	# Email notification setup
	TO_EMAIL="your-email@example.com"
	SUBJECT="Log Alert: High Error Counts in app.log"

	# Check if the log file exists
	if [[ ! -f "$LOG_FILE" ]]; then
	  echo "Error: Log file not found at $LOG_FILE"
	  exit 1
	fi

	# Count occurrences of each log level
	ERROR_COUNT=$(grep -c "ERROR" "$LOG_FILE")
	WARN_COUNT=$(grep -c "WARN" "$LOG_FILE")
	INFO_COUNT=$(grep -c "INFO" "$LOG_FILE")

	# Display the counts
	echo "Log Summary:"
	echo "ERROR: $ERROR_COUNT"
	echo "WARN: $WARN_COUNT"
	echo "INFO: $INFO_COUNT"

	# Send an email if any count exceeds 10
	if [[ "$ERROR_COUNT" -gt 10 || "$WARN_COUNT" -gt 10 || "$INFO_COUNT" -gt 10 ]]; then
	  MESSAGE="High error counts detected in $LOG_FILE\n\n"
	  MESSAGE+="ERROR: $ERROR_COUNT\n"
	  MESSAGE+="WARN: $WARN_COUNT\n"
	  MESSAGE+="INFO: $INFO_COUNT\n"
	  echo -e "$MESSAGE" | mail -s "$SUBJECT" "$TO_EMAIL"
	fi

	exit 0
	
	```

## Commands to Run the Script

**To create the file** 

	```bash
	nano process_logs.sh
	```

**To run the script** 

	```bash
	chmod +x process_logs.sh
	./process_logs.sh
	```

## Emial Notificaiton Configuration


**Install Required Tools and Dependencies**

- Mailutils: to send email notifications
- Postfix: For email relaying.

	```bash
	sudo apt update
	`
	sudo apt install mailutils

	sudo apt install postfix mailutils -y

	sudo apt install postfix libsasl2-modules

	```
#### Mailutils: Configuration

When asked to Configure Select 

**Internet with smarthost from the menu**

- **Purpose:** Configures the server to send outgoing mail via another mail server (a smarthost), while still being able to receive mail directly via SMTP or fetchmail.

When to choose:

If your ISP or hosting provider requires you to relay mail through their mail server.
You need outgoing mail to pass through another server for compliance or filtering.
Example use case: Relaying outgoing mail through smtp.yourprovider.com.

**What to Enter for "System Mail Name"**

- **For Personal Use:**
If this is for testing or personal use, you can enter the hostname of your server. For example:

	```bash
	your-server-name.local
	```
	
You can find your system’s hostname by running:

	```bash
	hostname -f
	```

- **For a Domain Name:**

If you have a custom domain (e.g., example.com), you can use it. For instance:

	```bash
	example.com
	```
	
- **If You Don't Have a Domain:**

You can use a generic placeholder, such as:

	```bash
	localhost
	```
	
or

	```bash
	localdomain
	```

#### Configure Postfix for a Relayhost

Edit the Postfix configuration file (/etc/postfix/main.cf) to include an external SMTP server (like Gmail):

	```bash
	sudo nano /etc/postfix/main.cf
	```
	
**Updated File**

	```bash
	# General settings
	smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)
	biff = no
	append_dot_mydomain = no
	readme_directory = no
	compatibility_level = 3.6

	# TLS parameters
	smtpd_tls_cert_file = /etc/ssl/certs/ssl-cert-snakeoil.pem
	smtpd_tls_key_file = /etc/ssl/private/ssl-cert-snakeoil.key
	smtpd_tls_security_level = may
	smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
	smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

	# SASL parameters
	smtp_sasl_auth_enable = yes
	smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
	smtp_sasl_security_options = noanonymous
	smtp_tls_security_level = encrypt

	# Relay host
	relayhost = [smtp.gmail.com]:587

	# Restrictions
	smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination

	# Networking
	myhostname = mashhood-HP-EliteBook-840-G5
	alias_maps = hash:/etc/aliases
	alias_database = hash:/etc/aliases
	mydestination = localhost, $myhostname, mashhood-HP-EliteBook-840-G5
	mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
	mailbox_size_limit = 0
	recipient_delimiter = +
	inet_interfaces = all
	inet_protocols = all
	```

#### Set Up Authentication

Create or edit the file /etc/postfix/sasl_passwd to store your Gmail credentials:

	```bash
	sudo nano /etc/postfix/sasl_passwd
	```

Add the following line (replace with your Gmail details):

	```bash
	[smtp.gmail.com]:587 your_email@gmail.com:your_app_password
	```

    Replace your_email@gmail.com with your Gmail address.
    Replace your_app_password with a Gmail App Password.

#### Secure and Hash the Credentials File

Run these commands to secure and hash the file:

	```bash
	sudo chmod 600 /etc/postfix/sasl_passwd
	sudo postmap /etc/postfix/sasl_passwd
	```
	
##### Restart Postfix

Restart the Postfix service to apply changes:

	```bash
	sudo systemctl restart postfix
	```

Test Email Sending

Use the mail command to send a test email:

	```bash
	echo "This is a test email body." | mail -s "Test Email Subject" your_email@gmail.com
	```
	
#### Check Email Sending Logs

Inspect the mail logs for errors. Run:


```bash
sudo tail -f /var/log/mail.log
```

When your script runs, any email-sending errors will appear here.

#### SASL Authentication Error

**Ensure Correct App Password**

    Gmail requires an App Password for third-party applications like Postfix when using your Gmail account for SMTP.

    **Steps to generate an App Password:**
        - Log in to your Gmail account.
        - Go to Google Account Security Settings.
        - Under "Signing in to Google", enable 2-Step Verification if not already done.
        - Click on "App Passwords" and generate a password for "Mail" and "Linux Computer."
        - Replace the password in /etc/postfix/sasl_passwd with the newly generated App Password:
	
	```bash
        [smtp.gmail.com]:587 your_email@gmail.com:your_app_password
	```
	
#### Rehash the Password File

After updating /etc/postfix/sasl_passwd, rehash the file and restart Postfix:

```bash
sudo postmap /etc/postfix/sasl_passwd
sudo systemctl restart postfix
```

**Enable "Allow Less Secure Apps" or Configure OAuth2**

    Gmail restricts less secure app access by default.
    If App Passwords are not an option, try enabling "Allow Less Secure Apps" from Gmail settings (not recommended for production environments):
        - Visit Less Secure App Access and toggle it on.
        - Note: Google may disable this feature at any time; App Passwords are the preferred method.

#### Test the SMTP Authentication

Send a test email and monitor the logs again:

```bash
echo "This is a test email body." | mail -s "Test Email Subject" your_email@gmail.com
```

## Question 5

**What is Amazon S3, and what are some of the key features of S3 storage? How would you implement security for sensitive data stored in S3, and what options would you use to optimize the cost for storing large amounts of data?**


### What is Amazon S3?

Amazon S3 (Simple Storage Service) is an object storage service offered by AWS that provides highly scalable, durable, and low-latency storage for a variety of data types. S3 allows you to store and retrieve any amount of data from anywhere on the web. It is commonly used for backup and restore, archiving, big data analytics, disaster recovery, content distribution, and more.

#### Key Features of Amazon S3

- **Scalability:** S3 scales automatically to handle an infinite amount of data and storage needs. You can store virtually any amount of data without worrying about capacity planning.

- **Durability:** S3 provides 99.999999999% (11 9's) durability over a given year, ensuring your data is safe from loss.

- **High Availability:** S3 is designed to deliver 99.99% availability over a given year, meaning your data will always be available when needed.

- **Security:** S3 integrates with AWS IAM (Identity and Access Management), enabling you to control access to your data through policies, bucket ACLs, and encryption.

- **Storage Classes:** S3 offers different storage classes such as Standard, Infrequent Access (IA), One Zone-IA, and Glacier to optimize costs based on data access patterns.

- **Versioning:** S3 supports versioning of objects, allowing you to preserve, retrieve, and restore every version of every object in a bucket.

- **Lifecycle Policies:** S3 allows you to set rules that automate the transition of objects between storage classes or delete them after a specified time.

- **Access Control:** S3 allows fine-grained control over who can access and perform actions on your data using IAM policies, ACLs, and bucket policies.

- **Event Notifications:** S3 can trigger notifications when certain events occur (e.g., an object is uploaded, deleted, etc.) to other AWS services like Lambda, SNS, or SQS.

### Securing Sensitive Data Stored in Amazon S3

There are several strategies and AWS features available to ensure sensitive data stored in S3 is protected from unauthorized access:

1. **Use Encryption**

   - **Server-Side Encryption (SSE):**
     - `**SSE-S3:**` S3 automatically encrypts your data using AES-256 when uploading and decrypts when downloading (no management required).
     - `**SSE-KMS:**` Uses AWS Key Management Service (KMS) to manage encryption keys, providing more control over key management, auditing, and permissions.
     - `**SSE-C:**` Allows you to manage your own encryption keys. S3 will use the provided key to encrypt/decrypt data.
   - **Client-Side Encryption:** Encrypt the data before uploading to S3. This ensures the data is encrypted on the client side before it reaches S3.

2. **Enable Bucket Versioning**

   Enable versioning in the bucket to keep multiple versions of your data, allowing you to restore earlier versions of an object. This can be crucial in recovering from accidental deletions or overwrites.

3. **Use Bucket Policies and Access Control Lists (ACLs)**

   - **Bucket Policies:** Use bucket policies to specify who can access the bucket and the actions they can perform. Policies are JSON documents that allow you to define permissions based on factors like IP address, time, and request origin.
   - **ACLs (Access Control Lists):** Use ACLs to grant access to specific AWS accounts or users for individual objects. ACLs allow you to define read/write permissions on an object level.

4. **Use IAM (Identity and Access Management) Policies**

   Define detailed IAM policies to control user and service permissions. Ensure only authorized users or services can access your S3 data. Use principles of least privilege when assigning permissions.

   IAM roles can also be used to allow EC2 instances, Lambda functions, or other AWS services to interact with S3 while maintaining access controls.

5. **Enable MFA (Multi-Factor Authentication) for Deletions**

   Protect your S3 buckets from accidental or malicious deletions by requiring MFA for delete operations. You can configure this using MFA Delete, which requires an MFA token to delete objects or versioned data.

6. **Use Access Points for Fine-Grained Access Control**

   S3 Access Points allow you to create unique access configurations for specific data sets within an S3 bucket. These access points simplify the process of managing access to large datasets or shared buckets, ensuring that only the right users and applications have the correct permissions.

7. **Enable Logging and Monitoring**

   - **S3 Access Logs:** Enable access logging to capture detailed records of requests made to your bucket, including the IP address, time, request type, and more. You can store these logs in a separate S3 bucket for auditing.
   - **AWS CloudTrail:** Use AWS CloudTrail to log API calls made to S3 and other AWS services. CloudTrail helps track who accessed your S3 resources, when, and what actions were performed.

8. **Use VPC Endpoints for Secure Connections**

   Use VPC endpoints to securely connect to S3 without exposing your traffic to the public internet. VPC endpoints ensure that data is transferred within AWS’s network, reducing the risk of data interception.

9. **Use S3 Block Public Access**

   Ensure that your S3 bucket and objects are not publicly accessible by using the S3 Block Public Access feature. This feature blocks public access at both the bucket and account levels, preventing any accidental exposure of sensitive data.

10. **Data Integrity Checks (MD5)**

    Ensure data integrity by verifying checksums. You can configure S3 to return an MD5 checksum upon data upload. When you download the object, you can validate the integrity by comparing the returned checksum.

### Optimizing Cost for Storing Large Amounts of Data

To optimize costs while storing large amounts of data in S3, you can consider the following options:

1. **Use Storage Classes**

   - `**S3 Standard:**` Best for frequently accessed data. It's optimized for high durability, availability, and performance.
   - `**S3 Infrequent Access (IA):**` For data that is not accessed frequently but needs to be available quickly when needed. This class is cheaper than the Standard class but incurs retrieval costs.
   - `**S3 One Zone-IA:**` A more cost-effective option than IA, but it stores data in a single availability zone, making it less durable.
   - `**S3 Glacier and Glacier Deep Archive:**` For archiving data that is rarely accessed. Glacier is low-cost storage for archival data with retrieval times ranging from minutes to hours, and Glacier Deep Archive is even cheaper with longer retrieval times (up to 12 hours).

2. **Lifecycle Policies**

   Use lifecycle policies to automatically transition data between storage classes based on age or access patterns. For example, you can move data to Glacier after 30 days of inactivity, or delete data after a certain time.

3. **Data Compression**

   Compress data before uploading it to S3 to reduce storage costs. S3 does not compress your data by default, so manually compressing large files can significantly reduce storage costs.

4. **Object Expiry**

   Use object expiration policies to automatically delete objects after a specified period of time. This is particularly useful for logs, temporary files, and other non-essential data.

5. **Optimize Uploads with Multipart Upload**

   For large objects, use Multipart Uploads to upload parts of an object concurrently, which reduces upload time. It also allows you to upload only the parts of an object that have changed, saving on bandwidth and storage costs.



## References

https://medium.com/@RoussiAbdelghani/optimizing-java-base-docker-images-size-from-674mb-to-58mb-c1b7c911f622


https://docs.oracle.com/en/java/javase/17/


