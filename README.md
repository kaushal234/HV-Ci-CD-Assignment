# CI/CD Pipeline – Flask & MongoDB on AWS EC2

A CI/CD pipeline built with **Jenkins** that tests a Python Flask application,
packages it into a Docker image, pushes it to **Amazon ECR**, deploys it as a
container on an **EC2** instance, verifies it via a `/health` endpoint, and
sends a customized email notification reporting success or failure.

## Architecture

```
Developer push (main)
        |
        v
   Jenkins pipeline
        |
   Checkout -> Install -> Test (pytest) -> Build (Docker) -> Push to ECR
        |
        v
   Deploy to EC2 (SSH): pull image, replace running container
        |
        v
   Verify (/health) -> Email notification (success / failure)
```

## Pipeline stages

The pipeline (`Jenkinsfile`) runs, in order:

1. **Checkout** – pull latest source from `main`.
2. **Install** – install dependencies from `requirements.txt`.
3. **Test** – start a temporary MongoDB container and run the `pytest` suite. The pipeline stops here if any test fails.
4. **Build** – build the Docker image, tagged with the short Git commit SHA.
5. **Push to ECR** – authenticate to ECR and push the tagged image.
6. **Deploy to EC2** – SSH into the instance, pull the new image, stop and remove the old container, and run the new one on port 5000.
7. **Verify** – curl `/health` on the instance; a non-200 response fails the deploy.
8. **Notify** – send a customized success or failure email.

## Prerequisites (AWS resources, IAM permissions, EC2 setup)

Set up manually before the pipeline runs:

- **ECR repository** to hold the built images (`hv-ci-cd-assignment`).
- **App EC2 instance** (Amazon Linux 2023) with:
  - Docker installed and running.
  - An **IAM instance role** (`ec2-ecr-pull-role`) with the
    `AmazonEC2ContainerRegistryReadOnly` policy, so the instance can pull
    images from ECR without stored credentials.
  - A **security group** allowing inbound TCP **5000** (application) and
    **22** (SSH, used by the deploy step).
- **Jenkins EC2 instance** (Amazon Linux 2023) with Jenkins, Docker, Git,
  Java 21, and the AWS CLI installed. Security group allows **8080**
  (Jenkins UI) and **22**.
- An **IAM user** (`jenkins-ecr-user`) with the
  `AmazonEC2ContainerRegistryPowerUser` policy, whose access keys let the
  pipeline authenticate and push images to ECR.
- A **MongoDB Atlas** cluster with a database user and network access
  configured to allow the app EC2 instance to connect.

## Configuring the pipeline's required secrets

All sensitive values are stored in **Jenkins Credentials**
(*Manage Jenkins -> Credentials -> Global*) and are never committed to the repo:

| ID | Type | Purpose |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | Secret text | ECR authentication (from `jenkins-ecr-user`) |
| `AWS_SECRET_ACCESS_KEY` | Secret text | ECR authentication (from `jenkins-ecr-user`) |
| `app-ec2-ssh-key` | SSH private key | SSH into the app EC2 for deployment |
| `MONGO_URI` | Secret text | Application database connection string |
| `SECRET_KEY` | Secret text | Flask secret key |
| `gmail-smtp` | Username / password | SMTP account for email notifications |

The pipeline references these by ID (via `withCredentials` and `sshagent`),
so no secret value ever appears in the `Jenkinsfile`.

Email delivery is configured under *Manage Jenkins -> System*
(Extended E-mail Notification) using `smtp.gmail.com:465` with SSL and a
Gmail App Password supplied through the `gmail-smtp` credential.

## How the deploy step connects to EC2 (and why)

The pipeline uses the **SSH-based** deploy method. Jenkins runs on its own EC2
instance and connects to the application EC2 over SSH, using the Jenkins
`sshagent` step with the stored `app-ec2-ssh-key` private key. Once connected,
it authenticates to ECR (using the app instance's IAM role), pulls the new
image, stops and removes the previous container, runs the new container on
port 5000, and the following stage verifies `/health`.

**Why SSH:** it is simple and transparent, requires no additional AWS agent
configuration, and maps directly to the manual deployment steps below — which
makes the pipeline easy to reason about, reproduce, and debug.

## Reproducing a deployment manually

If the pipeline were unavailable, the same deployment can be performed by hand
by SSHing into the app EC2 and running the steps the pipeline automates:

```bash
ssh -i <key>.pem ec2-user@<app-ec2-ip>

# Authenticate to ECR (the instance's IAM role grants pull access)
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin <account>.dkr.ecr.us-east-1.amazonaws.com

# Pull the image (tagged with the Git commit SHA) and replace the container
docker pull <account>.dkr.ecr.us-east-1.amazonaws.com/hv-ci-cd-assignment:<tag>
docker stop student-app || true
docker rm student-app || true
docker run -d --restart unless-stopped --name student-app -p 5000:5000 \
  -e MONGO_URI='<mongo-uri>' \
  -e SECRET_KEY='<secret-key>' \
  <account>.dkr.ecr.us-east-1.amazonaws.com/hv-ci-cd-assignment:<tag>

# Verify the deployment
curl -f http://localhost:5000/health
```

## Health check

`GET /health` returns `200 {"status": "healthy"}` when the application can
reach MongoDB, and `500 {"status": "unhealthy"}` otherwise. The pipeline uses
this endpoint as the deploy-verification gate: a container that starts but
cannot reach its database is reported as a failed deployment.

## Screenshots

See the `Screenshots/` folder:

- `Success - Pipeline stages.png` – full successful pipeline run
- `Success - Email .png` – success notification email
- `Failed - Pipeline stage.png` – pipeline stopping at a failing test
- `Failed -Email.png` – failure notification email (names the failed stage)
- `Live Application View.png` – the running application on EC2
- `Health Endpoint.png` – the `/health` endpoint returning healthy