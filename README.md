<img width="1200" height="800" alt="AWS EB " src="https://github.com/user-attachments/assets/a212701d-5373-4395-9bac-98b590d77e0c" />


# Deploying a Dockerized Flask Cat GIF App on AWS with Elastic Beanstalk 

## Project Overview



Taking the already-containerised Flask Cat GIF app and deploying it to AWS using Elastic Beanstalk (EB), so the app is publicly reachable on the internet instead of just running locally.

Why this matters in the real world: knowing how to containerise an app is only half the job. Being able to actually get that container running on a public cloud platform, reachable by real users, is the other half. Elastic Beanstalk is a PaaS (Platform as a Service) that abstracts away a lot of the underlying provisioning (EC2, security groups, IAM roles), which makes it a great entry point for understanding what a cloud platform is doing on your behalf before tackling that same provisioning manually with Terraform.

## Problem



The Cat GIF app worked fine locally inside a Docker container, but a container running on a personal machine can't be shared with or accessed by anyone else. The app needed to be hosted somewhere publicly accessible, with minimal infrastructure management, and at zero/near-zero cost for a low-traffic hobby project.

## Solution


Publish the Docker image to Docker Hub as a public registry, then use AWS Elastic Beanstalk's single-container Docker platform to pull that image, run it on an EC2 instance, and expose it behind a public URL. All configured through the EB console, with a Single Instance environment tier (no load balancer) to stay within the AWS free tier.

## Architecture Breakdown



1. The Docker image is built locally and pushed to a **public Docker Hub repository**.
2. **Elastic Beanstalk** provisions an **EC2 instance** and installs Docker on it automatically.
3. EB reads **`Dockerrun.aws.json`**, which tells it which image to pull and which container port to expose.
4. The EC2 instance pulls the image from Docker Hub and runs it, mapping the container's internal port to a host port.
5. EB assigns a public **CNAME URL** (`<environment>.<region>.elasticbeanstalk.com`) that routes incoming traffic to the running container.
6. Application logs are streamed from the container to **CloudWatch/nginx logs**, viewable directly from the EB console.

## Prerequisites



- A Docker Hub account (free tier is fine)
- The Cat GIF app already built and working locally as a Docker image
- An AWS account with billing enabled (free tier eligible)
- AWS Console access (no CLI/Terraform used for this project — deployed manually via console)

## Tools & Services Used



- **Docker Hub** — public registry hosting the built image
- **AWS Elastic Beanstalk (EB)** — PaaS that provisions and manages the EC2 instance, security group, and Docker runtime
- **Amazon EC2** — the underlying virtual machine EB provisions to actually run the container
- **Amazon S3** — used internally by EB to store the uploaded application bundle
- **Amazon CloudWatch** *(via EB logs)* — collects and displays container/application logs

## Preparation



This project doesn't use Terraform. Elastic Beanstalk was configured directly through the AWS Console. The only file involved (besides the existing Dockerfile/app code) is:

- **`Dockerrun.aws.json`** — an EB-specific config file that tells Elastic Beanstalk which Docker image to pull (`Image.Name`), whether to always re-pull the latest version (`Image.Update`), which container port maps to which host port (`Ports`), and where application logs live inside the container (`Logging`).

Because the EB console's "Local file" uploader only accepts `.zip`, `.war`, or `.jar` files, `Dockerrun.aws.json` needs to be compressed into a `.zip` archive (with the file sitting at the root of the archive, not nested in a subfolder) before it can be uploaded.

## Steps



### **1. Push the Docker image to Docker Hub**

Logged in with `docker login`, then built the image explicitly for the `linux/amd64` platform (since EC2 instances default to `amd64`, and Docker Desktop on Apple Silicon Macs defaults to `arm64` unless told otherwise) and pushed it to a public Docker Hub repository so Elastic Beanstalk can pull it later.

<img width="905" height="673" alt="image-10" src="https://github.com/user-attachments/assets/026541b2-9428-434c-85fb-fc13352f5b08" />


### **2. Prepare the Dockerrun.aws.json file**

Edited the `Image.Name` field to point to the pushed Docker Hub repository, confirmed the container port mapping matched the Flask app's listening port (5000), then zipped the file on its own so it could be uploaded through the EB console.

<img width="767" height="457" alt="image-11" src="https://github.com/user-attachments/assets/9fc060a4-904a-4aad-9358-9f5a69198b3b" />


### **3. Create the Elastic Beanstalk Application**

In the AWS Console, created a new Elastic Beanstalk Application to act as the logical namespace/project container for the environment that would actually run the app.

<img width="1418" height="330" alt="image-12" src="https://github.com/user-attachments/assets/99bffcbb-7532-46ac-bd7e-08175797662a" />


### **4. Configure and create the Environment**

Created a new **Web server environment**, selected **Docker** as the platform, and chose the **Single Instance** environment tier (rather than Load Balanced/Auto Scaling) with a **t2.micro/t3.micro** instance type to stay within the AWS free tier and avoid the ongoing cost of a Load Balancer.

<img width="1136" height="277" alt="image-13" src="https://github.com/user-attachments/assets/e2888c18-403d-4923-9f87-03168b43b2b7" />

<img width="1124" height="466" alt="image-14" src="https://github.com/user-attachments/assets/9b57f0e5-f24b-4246-a561-a16ba68b9cfb" />



### **5. Upload the application code and deploy**

Uploaded the zipped `Dockerrun.aws.json` as the application source bundle and launched the environment. EB provisioned the EC2 instance, installed Docker, pulled the image from Docker Hub, and started the container automatically.


## Validation & Testing


#### **1. Confirm environment health**

Checked the Elastic Beanstalk dashboard for a green health status, confirming the instance launched successfully and the container is running without errors.

<img width="1170" height="308" alt="image-15" src="https://github.com/user-attachments/assets/38e73284-c30d-4012-98d7-78f14b4e84da" />


#### **2. Load the app via the public URL**

Opened the environment's public CNAME URL in a browser and refreshed multiple times to confirm the Flask app is reachable over the internet and serving a random cat GIF on each load.

```
http://<environment-name>.<region>.elasticbeanstalk.com
```
<img width="850" height="627" alt="image-16" src="https://github.com/user-attachments/assets/468a2091-c8ea-4328-a4b1-8a376e3fdfbd" />


#### **3. Review deployment logs**

Pulled the last 100 lines of logs directly from the EB console to confirm the container started cleanly with no errors.
