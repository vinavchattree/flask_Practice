CI/CD Assignment



This project demonstrates a complete CI/CD pipeline for a Flask application using GitHub Actions, Docker, Amazon ECR, and an AWS EC2 instance.

The pipeline automatically:

1. Runs Python tests.
2. Builds a Docker image.
3. Tags the image using the Git commit SHA.
4. Pushes the image to Amazon ECR.
5. Deploys the exact image to an EC2 instance.
6. Performs a deployment health check.
7. Sends a customized email notification for success or failure.


(Testing without Git
- I forked the repository and  tested the app on my VS code

   <img width="940" height="470" alt="image" src="https://github.com/user-attachments/assets/94e913cd-da70-4a82-98cca5489acde7dd" />

    It started successfully

<img width="940" height="463" alt="image" src="https://github.com/user-attachments/assets/8402a999-6083-4982-a337-4714111c51aa" />

 - Created a cluster for mongodb. Saved credentials in env file and added health point part here :

   from flask import Flask, render_template, request, redirect, url_for,jsonify

  @app.route('/health', methods=['GET'])
def health_check():
    try:      
         mongo.cx.admin.command('ping') # Check if the MongoDB server is reachable

         mongo.db.students.find_one()  # Attempt to find one document in the 'students' collection

    except Exception as e:
        print(e)
        return jsonify({"status": "unhealthy", "error": "Not able to connect"}), 503
    return jsonify({"status": "healthy","mongodb":"connected","student db":"accessible"})

    <img width="940" height="347" alt="image" src="https://github.com/user-attachments/assets/209f5ac4-6bee-41de-a8bc-9a4251caf77c" />



- Then I build image for app and tested it on desktop docker

  <img width="940" height="314" alt="image" src="https://github.com/user-attachments/assets/e7fe86c8-14ee-4e3e-b226-c84bdb83f5c6" />
  <img width="940" height="370" alt="image" src="https://github.com/user-attachments/assets/596ec6b2-208a-4775-beda-2513fd4b011e" />

<img width="940" height="358" alt="image" src="https://github.com/user-attachments/assets/6f3a4977-af46-41fb-b722-b60e4f9ba72e" />


- I then created ECR repository
  <img width="940" height="423" alt="image" src="https://github.com/user-attachments/assets/6937da80-99ac-4be7-9d1a-16643840aee7" />

- Tried to push docker image to ECR
   
- Created IAM role for EC2 so that image can be pulled from ECR

  <img width="940" height="365" alt="image" src="https://github.com/user-attachments/assets/2533f9bd-5a7b-415f-a0be-3891b4fd75ff" />

- Tried to deploy on EC2 using docker
<img width="940" height="331" alt="image" src="https://github.com/user-attachments/assets/1ac30c8f-7cc3-48a5-8457-9063e31dddc0" />)




## CI/CD Workflow

The workflow is triggered whenever code is pushed to the `main` branch.

```yaml
on:
  push:
    branches:
      - main
```

### 1. Checkout

GitHub Actions checks out the repository using:

```yaml
uses: actions/checkout@v4
```

### 2. Python Setup

Python 3.11 is installed using:

```yaml
uses: actions/setup-python@v5
```

### 3. Install Dependencies

Dependencies are installed from `requirements.txt`:

```bash
pip install -r requirements.txt
```

### 4. Run Tests

The application tests are executed using:

```bash
pytest
```

The test environment uses the `MONGO_TEST_URI` GitHub repository secret.

The test database is kept separate from the production database used by the EC2 deployment.

### 5. Docker Build

After successful tests, the Docker image is built:

```bash
docker build -t flask-practice:${{ github.sha }} .
```

The GitHub commit SHA is used as the Docker image tag.

For example:

```text
flask-practice:14be177b0ee213e4ac6556a76d8719169dc0482b
```

Using the commit SHA makes every Docker image traceable to the exact source-code commit that produced it.

---

## Amazon ECR

The Docker image is pushed to:

```text
691317217805.dkr.ecr.ap-south-1.amazonaws.com/flask-practice
```

The workflow first authenticates to AWS using GitHub repository secrets:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

The AWS region is:

```text
ap-south-1
```

The image is then tagged with the ECR registry and pushed:

```bash
docker tag flask-practice:$IMAGE_TAG \
  $REGISTRY/$REPOSITORY:$IMAGE_TAG

docker push $REGISTRY/$REPOSITORY:$IMAGE_TAG
```

---

## EC2 Deployment

After the image is successfully pushed to ECR, GitHub Actions connects to the EC2 instance using SSH.

The following GitHub Secrets are used:

```text
EC2_HOST
EC2_USER
EC2_PRIVATE_KEY
```

On EC2, the deployment performs the following operations:

### ECR Login

```bash
aws ecr get-login-password --region ap-south-1 | \
docker login --username AWS --password-stdin \
691317217805.dkr.ecr.ap-south-1.amazonaws.com
```

### Pull the New Image

The exact Git SHA image is pulled:

```bash
docker pull $IMAGE
```

### Stop Existing Container

```bash
docker stop flask-app || true
```

### Remove Existing Container

```bash
docker rm flask-app || true
```

The `|| true` allows the first deployment to continue when the container does not yet exist.

### Start the New Container

```bash
docker run -d \
  --name flask-app \
  --env-file /home/ubuntu/flask.env \
  -p 5000:5000 \
  $IMAGE
```

The production MongoDB URI and other application configuration are stored in:

```text
/home/ubuntu/flask.env
```

This keeps production configuration separate from the GitHub test environment.

---

## Deployment Health Check

After starting the container, the workflow waits for the application to start and checks:

```bash
curl --fail http://localhost:5000/health
```

The Flask application exposes the `/health` endpoint to verify MongoDB connectivity and application health.

If the endpoint returns a successful HTTP response, the deployment succeeds.

If the endpoint returns an error such as HTTP 500, the GitHub Actions deployment step fails.

This prevents a container that starts but is not functioning correctly from being reported as a successful deployment.

---

## Email Notifications

The pipeline sends customized Gmail notifications using SMTP.

SMTP configuration:

```text
Server: smtp.gmail.com
Port: 465
SSL: enabled
```

Email credentials are never hardcoded in the workflow.

The following GitHub Secrets are used:

```text
MAIL_USERNAME
MAIL_PASSWORD
MAIL_FROM
MAIL_TO
```

`MAIL_PASSWORD` contains a Gmail App Password.

### Success Email

The success email has a subject similar to:

```text
[SUCCESS] CI/CD-HeroVired Deployment
```

It includes:

- Success status
- Repository
- Branch
- Git commit SHA
- Docker image pushed to ECR
- EC2 deployment target
- Link to the GitHub Actions pipeline run

### Failure Email

The failure email has a subject similar to:

```text
[FAILED] CI/CD-HeroVired Deployment
```

It includes:

- Failure status
- Repository
- Branch
- Git commit SHA
- Failed stage
- Stage results
- Docker image information
- EC2 target
- Link directly to the GitHub Actions pipeline run

The pipeline tracks the major stages as:

```text
test
build
push
deploy
```

For example:

```text
Failed Stage: deploy
```

This makes it possible to identify where the pipeline failed without manually searching through the GitHub Actions interface.

---

## GitHub Repository Secrets

The pipeline uses GitHub Actions repository secrets for sensitive information.

### MongoDB

```text
MONGO_TEST_URI
```

Used for CI tests.

### AWS

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

Used by GitHub Actions to authenticate with AWS.

### EC2

```text
EC2_HOST
EC2_USER
EC2_PRIVATE_KEY
```

Used for SSH deployment.

### Email

```text
MAIL_USERNAME
MAIL_PASSWORD
MAIL_FROM
MAIL_TO
```

Used for Gmail SMTP notifications.

No passwords, private keys, MongoDB credentials, or email credentials are stored directly in the workflow file.

---

## IAM Permissions

The GitHub Actions IAM user requires permissions for the ECR operations performed by the workflow, including authentication and pushing images.

The EC2 instance also requires permission to retrieve an ECR authorization token and pull images from the ECR repository.

The workflow separates the two authentication contexts:

```text
GitHub Actions IAM credentials
        |
        +--> Push image to ECR

EC2 IAM role
        |
        +--> Pull image from ECR
```

---

## Failure Handling

The pipeline is designed to stop when an important stage fails.

Failure email
```

A successful deployment requires the complete pipeline to finish successfully, including the health check.

---

## Image Versioning

Docker images are tagged using the GitHub commit SHA:

```text
flask-practice:<commit-sha>
```

This provides immutable identification of the application version being deployed.

For example:

```text
flask-practice:14be177b0ee213e4ac6556a76d8719169dc0482b
```

The same SHA is used when:

1. Building the image.
2. Tagging the ECR image.
3. Pushing the image.
4. Pulling the image on EC2.

This ensures that the image tested and pushed by the pipeline is the exact image deployed to EC2.

---

This provides an automated CI/CD workflow from source-code commit through testing, containerization, image storage, EC2 deployment, verification, and notification.




   










   


