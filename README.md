# Shuffleboard Scoreboard Application

This README provides instructions on how to deploy and run the Shuffleboard Scoreboard application both locally using Docker Compose and on Google Cloud Run.

## Prerequisites

Before you begin, ensure you have the following installed:

* **Docker:** For containerization. Install from [https://docs.docker.com/get-docker/](https://docs.docker.com/get-docker/)
* **Docker Compose:** Usually installed with Docker Desktop. If not, follow instructions here: [https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)
* **Google Cloud CLI (gcloud):** For deploying to Google Cloud Run. Install from [https://cloud.google.com/sdk/docs/install](https://cloud.google.com/sdk/docs/install)
* **Google Cloud Project:** You need a Google Cloud Project set up.
* **Enabled Cloud Run API:** Ensure the Cloud Run API is enabled in your Google Cloud Project.
* **Configured gcloud CLI:** Make sure you have authenticated and configured the `gcloud` CLI to point to your Google Cloud Project. You can do this using `gcloud auth login` and `gcloud config set project YOUR_PROJECT_ID`.

## Local Deployment using Docker Compose

This method uses Docker Compose to build and run both the frontend and backend containers on your local machine.

1.  **Navigate to the project root directory.** This is the directory containing the `docker-compose.yml` file and the `frontend` and `backend` subdirectories.

2.  **Build and start the containers:**
    ```bash
    docker-compose up --build
    ```
    The `--build` flag ensures that the Docker images for both the frontend and backend are built if they haven't been already, or if the Dockerfiles have changed.

3.  **Access the application:**
    * **Frontend:** Once the containers are running, the frontend should be accessible in your web browser at `http://localhost:3000`.
    * **Backend:** The backend will be running at `http://localhost:5000`. The frontend is configured to communicate with the backend at this address (within the Docker network).

4.  **Stop the containers:**
    To stop the running containers, navigate to the project root directory in your terminal and run:
    ```bash
    docker-compose down
    ```
    This will stop and remove the containers created by `docker-compose up`.

## Deployment to Google Cloud Run

This section outlines the steps to deploy the frontend and backend as separate services on Google Cloud Run.

### 1. Build and Push Docker Images to Google Container Registry (GCR)

**For the Frontend:**

1.  Navigate to the `frontend` directory:
    ```bash
    cd frontend
    ```

2.  Build the Docker image and tag it with your Google Container Registry address:
    ```bash
    docker build -t gcr.io/$GOOGLE_CLOUD_PROJECT/scoreboard-frontend .
    ```
    Replace `$GOOGLE_CLOUD_PROJECT` with your actual Google Cloud Project ID.

3.  Push the Docker image to GCR:
    ```bash
    docker push gcr.io/$GOOGLE_CLOUD_PROJECT/scoreboard-frontend
    ```

**For the Backend:**

1.  Navigate to the `backend` directory:
    ```bash
    cd ../backend
    ```

2.  Build the Docker image and tag it with your Google Container Registry address:
    ```bash
    docker build -t gcr.io/$GOOGLE_CLOUD_PROJECT/scoreboard-backend .
    ```
    Replace `$GOOGLE_CLOUD_PROJECT` with your actual Google Cloud Project ID.

3.  Push the Docker image to GCR:
    ```bash
    docker push gcr.io/$GOOGLE_CLOUD_PROJECT/scoreboard-backend
    ```

### 2. Deploy to Cloud Run

You can deploy to Cloud Run using either the Google Cloud Console or the `gcloud` CLI.

**Using the Google Cloud Console:**

1.  Go to the Cloud Run service in the Google Cloud Console.
2.  Click "Create Service".
3.  **For the Frontend Service:**
    * Select "Deploy one revision from an existing container image".
    * Enter the image URL: `gcr.io/$GOOGLE_CLOUD_PROJECT/scoreboard-frontend`.
    * Give your service a name (e.g., `scoreboard-frontend`).
    * Choose the region where you want to deploy.
    * Under "Ingress control", select "Allow all traffic".
    * Under "Container, Networking, Security", go to the "Container" tab and ensure the "Port" is set to `80`.
    * Click "Create".
4.  **For the Backend Service:**
    * Click "Create Service".
    * Select "Deploy one revision from an existing container image".
    * Enter the image URL: `gcr.io/$GOOGLE_CLOUD_PROJECT/scoreboard-backend`.
    * Give your service a name (e.g., `scoreboard-backend`).
    * Choose the region where you want to deploy.
    * Under "Ingress control", you might want to restrict this depending on your needs. For the frontend to access it, you'll likely need to allow traffic from the frontend's domain or make it publicly accessible for this basic setup.
    * Under "Container, Networking, Security", go to the "Container" tab and ensure the "Port" is set to `5000`.
    * Click "Create".

**Using the `gcloud` CLI:**

1.  **Deploy the Frontend Service:**
    ```bash
    gcloud run deploy scoreboard-frontend \
        --image gcr.io/$GOOGLE_CLOUD_PROJECT/scoreboard-frontend \
        --platform managed \
        --region YOUR_REGION \
        --port 80 \
        --allow-unauthenticated
    ```
    Replace `YOUR_REGION` with your desired Google Cloud region (e.g., `us-central1`).

2.  **Deploy the Backend Service:**
    ```bash
    gcloud run deploy scoreboard-backend \
        --image gcr.io/$GOOGLE_CLOUD_PROJECT/scoreboard-backend \
        --platform managed \
        --region YOUR_REGION \
        --port 5000 \
        --allow-unauthenticated # Consider restricting access later
    ```
    Replace `YOUR_REGION` with the same region you used for the frontend.

### 3. Configure Frontend to Connect to Backend on Cloud Run

After deploying both services, you'll need to configure your frontend to communicate with the backend's Cloud Run URL.

1.  **Get the Backend Service URL:** After the backend service is deployed on Cloud Run, it will be assigned a unique URL (e.g., `https://scoreboard-backend-abcdefg-uc.a.run.app`). You can find this URL in the Google Cloud Console or by using the `gcloud run services describe scoreboard-backend --format='value(status.url)'` command.

2.  **Update the Frontend Code:** In your `frontend/src/components/Scoreboard.tsx` file, you need to update the `socket.io` connection URL to point to your backend's Cloud Run URL:

    ```typescript
    // frontend/src/components/Scoreboard.tsx
    import React, { useState, useEffect } from 'react';
    import './Scoreboard.css';
    import { io, Socket } from 'socket.io-client';

    // Replace 'http://localhost:5000' with your backend Cloud Run URL
    const BACKEND_URL = 'YOUR_BACKEND_CLOUD_RUN_URL';

    // ...

    useEffect(() => {
      const newSocket = io(BACKEND_URL);
      setSocket(newSocket);

      // ... rest of your useEffect code
    }, []);

    // ...
    ```

    Replace `YOUR_BACKEND_CLOUD_RUN_URL` with the actual URL of your deployed backend service. **Make sure to use `https://` if your Cloud Run service is configured for HTTPS (which is the default).**

3.  **Rebuild and Redeploy the Frontend:** After updating the backend URL in your frontend code, you need to rebuild the Docker image and redeploy the frontend service to Cloud Run following the steps in section 1 and 2 for the frontend.

### Running Frontend and Backend on the Same Cloud Run Instance (Considerations)

While technically possible to run multiple processes within a single Cloud Run container, it's generally **not recommended** for this type of application due to:

* **Complexity:** Managing two separate applications within one container adds complexity to the Dockerfile, entrypoint script, and overall management.
* **Scalability:** Cloud Run scales instances based on container resource usage and requests. Running them together might lead to inefficient scaling if one service is more resource-intensive than the other.
* **Resource Isolation:** Running them separately provides better resource isolation and prevents issues in one service from directly impacting the other.
* **Lifecycle Management:** Deploying and updating them separately is cleaner and less prone to errors.

However, if you still want to explore this (e.g., for very minimal resource usage in a simple scenario), you would need to:

1.  Create a single Dockerfile that builds both the frontend and backend.
2.  Use a process manager (like `supervisord`) within the container to run both the frontend (likely serving static files with Nginx) and the backend (Flask application) simultaneously.
3.  Configure the Nginx to serve the built frontend files.
4.  Expose the necessary port (likely 80 for the frontend).

This approach is more involved and less aligned with the best practices for containerized microservices on platforms like Cloud Run. It's generally better to keep them as separate, independently deployable services.

By following these steps, you should be able to deploy your Shuffleboard Scoreboard application both locally using Docker Compose and as separate services on Google Cloud Run. Remember to adjust configurations (like regions, ports, and access control) according to your specific needs.