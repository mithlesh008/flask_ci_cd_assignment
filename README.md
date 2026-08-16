# Student Registration System

A simple **Flask** web application to manage student records with **MongoDB** as the backend database. Users can **add, view, update, and delete** student details.

---

## Features

* List all students on the home page
* Add a new student
* Update existing student details
* Delete a student with confirmation
* Simple and responsive UI using Bootstrap

---

## Tech Stack

* **Backend:** Python, Flask
* **Database:** MongoDB (via Flask-PyMongo)
* **Frontend:** HTML, Jinja2 templates, Bootstrap 5
* **Environment Variables:** Managed via `.env` file

---

# Flask CI/CD Pipeline with GitHub Actions

Repository: [flask_ci_cd_assignment](https://github.com/mithlesh008/flask_ci_cd_assignment)

This project demonstrates an end-to-end CI/CD pipeline for a Flask and MongoDB application using GitHub Actions, Docker, Amazon ECR, and an existing Amazon EC2 instance.

## Pipeline overview

On every push to the `main` branch, the workflow in `.github/workflows/ci-cd.yml` performs these steps in order:

1. Checks out the source code.
2. Installs Python dependencies from `requirements.txt`.
3. Runs the pytest suite. Build and deployment stop if a test fails.
4. Builds a Docker image tagged with the Git commit SHA.
5. Pushes the tagged image to the private Amazon ECR repository.
6. Connects to EC2 using SSH.
7. Pulls the image, replaces the running container, and starts it with `--restart unless-stopped`.
8. Polls `/health` and treats a failed health check as a failed deployment.
9. Sends a customized success or failure email.

The workflow file is:

```text
.github/workflows/ci-cd.yml
```

Only GitHub Actions is used for this assignment. No Jenkinsfile is required.

## Repository structure

```text
flask_ci_cd_assignment/
├── app.py
├── requirements.txt
├── test_app.py
├── templates/
├── Dockerfile
├── .dockerignore
├── .env.example
├── .gitignore
├── README.md
└── .github/
    └── workflows/
        └── ci-cd.yml
```

## Prerequisites

### Local machine

Install and configure:

* Git
* Python 3.11 or compatible Python version
* Docker
* AWS CLI
* SSH client

Clone the repository:

```bash
git clone https://github.com/mithlesh008/flask_ci_cd_assignment.git
cd flask_ci_cd_assignment
```

### AWS resources

The pipeline uses these AWS resources:

| Resource | Configuration used |
|---|---|
| AWS Region | `ap-south-1` |
| ECR repository | `mbagga-flask-repo` |
| EC2 instance | Existing Amazon Linux instance |
| Security group | Existing `mbagga-new` security group |
| Application port | TCP `5000` |
| SSH port | TCP `22` |
| EC2 SSH user | `ec2-user` |

If your actual Region, repository name, or EC2 username is different, use your actual values in GitHub Variables and this README.

### ECR repository

The ECR repository is private. The repository must exist before the workflow runs:

```text
ECR repository: mbagga-flask-repo
Repository type: Private
Region: ap-south-1
```

The GitHub Actions identity needs permission to authenticate and push images. The EC2 instance role needs permission to authenticate and pull images.

### EC2 instance

The existing EC2 instance must have:

* Docker installed and running.
* AWS CLI installed.
* `curl` installed.
* A public IP address or another reachable address for SSH deployment.
* An IAM instance role attached.
* The `AmazonEC2ContainerRegistryReadOnly` policy, or an equivalent scoped ECR pull policy, attached to the EC2 role.
* The `mbagga-new` security group attached.

Verify Docker on EC2:

```bash
docker --version
sudo systemctl is-active docker
docker run --rm hello-world
```

Verify the EC2 IAM role:

```bash
aws sts get-caller-identity
```

The returned ARN should show the EC2 instance role. Test ECR authentication from EC2:

```bash
export AWS_REGION='ap-south-1'
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export ECR_REPOSITORY='flask-practice'
export ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

aws ecr get-login-password --region "$AWS_REGION" \
  | docker login \
      --username AWS \
      --password-stdin "$ECR_REGISTRY"
```

Expected output:

```text
Login Succeeded
```

### Security group

The existing `mbagga-new` security group must allow:

| Protocol | Port | Source | Purpose |
|---|---:|---|---|
| TCP | 5000 | Approved test IP/network | Access the Flask application |
| TCP | 22 | SSH source permitted for the lab | GitHub Actions SSH deployment |

Port 22 was used for this lab because the deployment connects to the existing EC2 instance using SSH and the existing private key. Opening SSH from anywhere is not recommended for production; restrict it to an approved source IP or use SSM in a production setup.

## IAM permissions

### EC2 IAM role

The EC2 instance uses an attached IAM role with:

```text
AmazonEC2ContainerRegistryReadOnly
```

Purpose:

```text
EC2 → authenticate to private ECR → pull the Docker image
```

`AmazonSSMManagedInstanceCore` is not required for this implementation because the deployment method is SSH, not SSM.

### GitHub Actions IAM user

The GitHub Actions workflow uses the IAM access-key method rather than OIDC. A dedicated IAM user was created for GitHub Actions:

```text
github-actions-ecr-push
```

The user has a scoped inline policy that allows ECR login and pushing images to the `flask-practice` repository. At minimum, the policy permits:

```text
ecr:GetAuthorizationToken
ecr:BatchCheckLayerAvailability
ecr:CompleteLayerUpload
ecr:InitiateLayerUpload
ecr:PutImage
ecr:UploadLayerPart
ecr:DescribeRepositories
```

The user access key is stored only in GitHub Secrets. It is never committed to the repository.

## GitHub Actions configuration

Open the repository:

```text
Settings → Secrets and variables → Actions
```

### Repository Variables

Add these under the **Variables** tab:

| Variable name | Value |
|---|---|
| `AWS_REGION` | `ap-south-1` |
| `ECR_REPOSITORY` | `mbagga-flask-repo` |
| `APP_PORT` | `5000` |
| `EC2_USER` | `ec2-user` |

### Repository Secrets

Add these under the **Secrets** tab:

| Secret name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access key ID for `github-actions-ecr-push` |
| `AWS_SECRET_ACCESS_KEY` | Secret access key for `github-actions-ecr-push` |
| `EC2_HOST` | Public IP or hostname of the existing EC2 instance |
| `EC2_SSH_KEY` | Complete contents of `mbagga-reproducer.pem` |
| `MONGO_URI` | MongoDB URI used by the deployed application |
| `MONGO_TEST_URI` | MongoDB URI used by pytest |
| `SMTP_HOST` | SMTP server hostname, for example `smtp.gmail.com` |
| `SMTP_PORT` | SMTP port, for example `465` |
| `SMTP_USERNAME` | SMTP username/email address |
| `SMTP_PASSWORD` | SMTP app password or approved SMTP password |
| `MAIL_FROM` | Sender email address |
| `MAIL_TO` | Recipient email address |

For `EC2_SSH_KEY`, copy the complete key including the first and last lines:

```text
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
```

Do not commit any of these values to GitHub files. Do not place a real MongoDB URI, AWS access key, SMTP password, or private key in `README.md`, `Dockerfile`, `.env`, or the workflow file.

## Docker and application configuration

The Docker image exposes port `5000` and includes a Docker health check for `/health`.

Build locally:

```bash
docker build -t flask-app:local .
```

Run locally with runtime environment injection:

```bash
docker run -d \
  --rm \
  --name flask-app-local \
  --restart unless-stopped \
  -p 5000:5000 \
  --env-file .env \
  flask-app:local
```

Verify the application:

```bash
curl -i http://127.0.0.1:5000/health
```

Expected response:

```json
{
  "status": "ok",
  "mongodb": "reachable"
}
```

The `.env` file is not copied into the image. It is supplied at runtime with `--env-file .env`. The deployment workflow creates the EC2 runtime environment file from the protected `MONGO_URI` GitHub Secret.

## How the deploy step connects to EC2

This implementation uses SSH.

The deploy job:

1. Writes the protected `EC2_SSH_KEY` secret to a temporary file on the GitHub Actions runner.
2. Uses `ssh-keyscan` to prepare the host key file.
3. Connects to EC2 using `EC2_USER` and `EC2_HOST`.
4. Copies the runtime `.env` file to `~/flask-cicd/.env` using `scp`.
5. Logs in to private ECR on EC2 using the EC2 instance role.
6. Pulls the Docker image tagged with the Git commit SHA.
7. Removes the previous `flask-app` container.
8. Starts the new container with `--restart unless-stopped`.
9. Polls `http://127.0.0.1:5000/health`.
10. Fails the deployment if the health endpoint does not return success.

SSH was selected because the existing EC2 instance was already reachable using `mbagga-reproducer.pem`, Docker was already installed, and SSH provides a direct way to run the required `docker pull`, `docker rm`, and `docker run` commands. SSM was not used for this implementation.

## Running the pipeline

The workflow runs automatically on every push to the `main` branch. It can also be started manually from the GitHub Actions page using **Run workflow**.

The workflow file is:

```text
.github/workflows/ci-cd.yml
```

The AWS credentials step uses the IAM access-key method:

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ env.AWS_REGION }}
```

The Docker image is tagged with:

```text
${{ github.sha }}
```

This makes every deployed image traceable to the source commit.

## Manual deployment if GitHub Actions is unavailable

The following commands reproduce the deployment manually from a machine with AWS CLI access and SSH access to EC2.

Set the deployment variables:

```bash
export AWS_REGION='ap-south-1'
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export ECR_REPOSITORY='flask-practice'
export ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
export EC2_HOST='<EC2_PUBLIC_IP_OR_HOSTNAME>'
export EC2_USER='ec2-user'
export SSH_KEY_FILE="$HOME/.ssh/mbagga-reproducer.pem"
export APP_PORT='5000'
export IMAGE_TAG='<GIT_COMMIT_SHA>'
export IMAGE_URI="${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"
```

Authenticate Docker to ECR from the deployment machine if required:

```bash
aws ecr get-login-password --region "$AWS_REGION" \
  | docker login \
      --username AWS \
      --password-stdin "$ECR_REGISTRY"
```

Create the EC2 application directory:

```bash
ssh -i "$SSH_KEY_FILE" \
  "$EC2_USER@$EC2_HOST" \
  'mkdir -p ~/flask-cicd'
```

Create or copy the runtime environment file to EC2. Do not commit this file:

```bash
scp -i "$SSH_KEY_FILE" \
  .env \
  "$EC2_USER@$EC2_HOST:flask-cicd/.env"
```

Set secure permissions:

```bash
ssh -i "$SSH_KEY_FILE" \
  "$EC2_USER@$EC2_HOST" \
  'chmod 600 ~/flask-cicd/.env'
```

Run the deployment on EC2:

```bash
ssh -i "$SSH_KEY_FILE" \
  "$EC2_USER@$EC2_HOST" \
  "AWS_REGION='$AWS_REGION' \
   ECR_REGISTRY='$ECR_REGISTRY' \
   IMAGE_URI='$IMAGE_URI' \
   APP_PORT='$APP_PORT' \
   bash -s" <<'REMOTE_SCRIPT'
set -Eeuo pipefail

aws ecr get-login-password \
  --region "$AWS_REGION" \
  | docker login \
      --username AWS \
      --password-stdin "$ECR_REGISTRY"

docker pull "$IMAGE_URI"
docker rm -f flask-app 2>/dev/null || true

docker run -d \
  --restart unless-stopped \
  --name flask-app \
  -p "${APP_PORT}:5000" \
  --env-file "$HOME/flask-cicd/.env" \
  "$IMAGE_URI"

for attempt in $(seq 1 30); do
  if curl --fail --silent --show-error \
      "http://127.0.0.1:${APP_PORT}/health"; then
    echo
    echo "Health check passed"
    exit 0
  fi
  sleep 2
done

echo "Health check failed"
docker ps -a --filter "name=flask-app"
docker logs --tail 100 flask-app
exit 1
REMOTE_SCRIPT
```

Verify the deployment:

```bash
curl -i "http://${EC2_HOST}:${APP_PORT}/health"
```

## Pipeline evidence

The assignment evidence includes a successful complete pipeline, the success email, an intentionally failed run, and the failure email.

### Screenshot placement

Create a `screenshots/` directory in the repository:

```bash
mkdir -p screenshots
```

Rename or copy your screenshots using these names:

```text
screenshots/01-successful-pipeline.png
screenshots/02-ecr-commit-sha.png
screenshots/03-ec2-container-health.png
screenshots/04-success-email.png
screenshots/05-intentional-failure.png
screenshots/06-failure-email.png
```

Before committing screenshots, redact:

* MongoDB usernames, passwords, and connection strings.
* AWS access keys.
* SSH private-key content.
* SMTP passwords or app passwords.
* Any other sensitive values.

Add the screenshots to this README by placing these lines under the Evidence section:

```markdown
![Successful GitHub Actions pipeline](screenshots/01-successful-pipeline.png)

![ECR image tagged with commit SHA](screenshots/02-ecr-commit-sha.png)

![EC2 container and health check](screenshots/03-ec2-container-health.png)

![Success email](screenshots/04-success-email.png)

![Intentional failed pipeline](screenshots/05-intentional-failure.png)

![Failure email](screenshots/06-failure-email.png)
```

If you are submitting screenshots separately through Vlearn, the screenshots do not have to be committed to GitHub. Keep the Evidence section and attach the screenshots separately. The repository should still contain this README.

## Troubleshooting

### ECR access denied in GitHub Actions

Check:

```text
AWS_ACCESS_KEY_ID is correct.
AWS_SECRET_ACCESS_KEY is correct.
AWS_REGION matches the ECR repository Region.
ECR_REPOSITORY matches the repository name.
The IAM user policy contains ECR push permissions.
```

### SSH connection failure

Check:

```text
EC2_HOST is the current EC2 public IP.
EC2_USER is ec2-user for Amazon Linux.
EC2_SSH_KEY contains the complete private key.
Security group mbagga-new allows TCP 22.
```

### Health check returns 503

On EC2 run:

```bash
docker ps -a
docker logs --tail 100 flask-app
curl -i http://127.0.0.1:5000/health
```

Check that the deployed runtime `.env` contains a valid `MONGO_URI` and that MongoDB Atlas allows the EC2 network address.

## Security notes

* Never commit `.env`, real MongoDB URIs, `.pem` files, AWS access keys, or SMTP credentials.
* Use GitHub Secrets for sensitive values.
* Use a dedicated IAM user for GitHub Actions and scope its ECR permissions to the assignment repository.
* Restrict SSH access after the lab if port 22 was temporarily opened broadly.
* Rotate credentials if a secret is exposed.
* Do not include secret values in screenshots or README text.
