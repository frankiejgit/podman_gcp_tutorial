# Instructions 

### Overview

The great thing about using Podman, in addition to the benefits listed above, is that it is a drop-in replacement for docker. The experience in building, pushing and deploying container images is exactly the same. 

```bash
# To get our app to Artifact Registry
podman build --> podman tag --> podman push

# To deploy app in Cloud Run
gcloud run deploy
```
### Set-up

Let's start by installing __Podman__ by following these [instructions](https://podman.io/docs/installation) in their documentation. Once you have that set up, you can start using Podman by running:
```bash
podman machine init
podman machine start
```

To make things easier, let's declare some environment variables as well:
```bash
export REGION=<your-region-of-choice>    # e.g., us-central1
export PROJECT_ID=<your-project-id>
export REPO_NAME=<your-repo-name>        # e.g., podman-tutorial-repo
export IMAGE_PATH="${REGION}-us.pkg.dev/${PROJECT_ID}/${REPO_NAME}/basic-api:latest"
```

After this, assuming you already have a Google Cloud project (if not, follow these [steps](https://developers.google.com/workspace/guides/create-project)), we want to enable the services we plan on using. For this tutorial, we are only using **Artifact Registry** as our container image repository and **Cloud Run** to deploy our FastAPI app. 

 By default, Google Cloud disables all services to prevent unwanted actions or expenses on user accounts. You can enable them on the console by opening the side menu and navigating to __APIs & Services > Enabled APIs & Services__ or you can paste the following in your terminal:
 ```bash
gcloud services enable \
	artifactregistry.googleapis.com \
	run.googleapis.com
 ```

As a user, you need to ensure you have the following IAM roles assigned to you to complete the tutorial:
- __Artifact Registry Writer__ to upload/download artifacts
- __Cloud Run Developer__ to create/update services

### Create Artifact Registry repo

Podman uses standard OCI images, so when creating the repository , use the `docker` format (making it even easier to adopt `podman`).

```bash
gcloud artifacts repositories create $REPO_NAME \
	--repository-format=docker \
	--location=$REGION \
	--description="Repo for a Podman with GCP tutorial"
```

There you have it, now we can build and push our image to the repo.
### Build a sample container image

Easiest way to start is to use this existing repository by running `git clone ...` on your terminal. If you want to create the project from scratch you can follow along.

```bash
# Create a local directory
mkdir podman_gcp_tutorial
cd podman_gcp_tutorial

# OPTIONAL - create a virtual environment using `uv`
uv init
uv venv .venv
uv pip install "fastapi[standard]" "uvicorn"
```

Create a file called `app.py` in the same directory and paste the code below (taken from FastAPI's [first steps tutorial](https://fastapi.tiangolo.com/tutorial/first-steps/) ):
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
	return {"message": "Hello World"}
```

You can test it locally by running `fastapi dev app.py` on your terminal. It will use `uvicorn` to run the API locally on port 8000. Enter the URL on your browser or enter `curl -X 'GET' 'http://127.0.0.1:8000/'` on your terminal.

Write the `requirements.txt` file:
```txt
fastapi
uvicorn
```

Write the Dockerfile/Containerfile for this app:
```Dockerfile
# Use a lightweight Python image
FROM python:3.12-slim

# Set working directory to /app
WORKDIR /app

# Copy requirements and install them
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy rest of application code
COPY app.py .

# Run the application with uvicorn, listening on port 8080
CMD gunicorn -k uvicorn.workers.UvicornWorker app:app --bind 0.0.0.0:$PORT
```

### Push image to Artifact Registry

Now that we have the Dockerfile and the Python script, let's build and push a container image to the repo we created. It is a simple process:

```bash
# Build image
podman build --platform linux/amd64 -t "basic-api" .

# Tag image
podman tag "basic-api" "${IMAGE_PATH}" 

# Push image
podman push "${IMAGE_PATH}"
```

We could skip the `tag` step and just build the image using the entire repo path from the start but it is a best practice to avoid using the repo path as a prefix in local environments.

Once we verify the image is in the repository, we can deploy it in Cloud Run with one simple step:
```bash
gcloud run deploy "basic-api" \
	--image="${IMAGE_PATH}" \
	--region="${REGION}" \
	--allow-unauthenticated
```

Wait until the command finishes running, it will give you the Cloud Run service URL (should look like this `https://basic-api-PROJECT_NUMBER.REGION.run.app`).

Now to test the service, let's do the same `curl` command with the new URL:
```bash
curl -X "GET" "`https://basic-api-PROJECT_NUMBER.REGION.run.app/"
```

You should see the same "Hello World" response from when we tested the app locally. If you enter the __Service URL__ on your browser appended with "/docs" you will also see the API documentation like this:

![[Screenshot 2025-11-06 at 3.23.25 PM.png]]

And that's it! You have successfully used Podman to push a container image to Artifact Registry and run it on Cloud Run.