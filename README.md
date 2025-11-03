# How to Run a Local, GPU-Accelerated LLM with AnythingLLM and Ollama on Debian (AMD ROCm Guide)

This guide documents the step-by-step process for setting up a fully private, offline, and GPU-accelerated Large Language Model (LLM) on a Debian 12 system using an AMD Radeon RX 7700 XT.

The goal is to run a local AI that can be "trained" on your own documents, with all processing handled by your on-premises hardware, ensuring no data ever leaves your network.

Getting this to work, especially with AMD GPU acceleration (ROCm), was a significant challenge involving driver issues, container networking, and model instability. This document outlines the final, stable configuration.

---

## Technology Stack

* **Operating System:** Debian 12 
* **GPU:** AMD Radeon RX 7700 XT
* **Container:** Docker & Docker Compose
* **AI Backend:** Ollama (using the official `ollama/ollama:rocm` Docker image)
* **GUI / Interface:** AnythingLLM (running in Docker)
* **AI Model:** `codellama:13b` 

---

## 1. Host System Prerequisites

Before launching any containers, your Debian system must be prepared to give Docker access to the GPU.

### 1.1. Install Docker & Docker Compose

First, ensure Docker and Docker Compose are installed and running.

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Install Docker packages:
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 1.2. Install AMD GPU Packages

The container needs to talk to your host system's GPU drivers. Install the necessary firmware and OpenCL packages.

```bash
sudo apt-get update
sudo apt-get install firmware-amd-graphics mesa-opencl-icd
```

### 1.3. Set User Permissions
**This is a critical step. Your user account must be in the render and video groups to access the GPU device files.**

```bash
sudo usermod -aG render,video $USER
```

**IMPORTANT: You must reboot your system after running this command for the group changes to take full effect.**

```bash
sudo reboot
```

## 2. The Solution: Docker Compose
The cleanest and most reproducible way to run this multi-container application is with Docker Compose. This single file defines the entire service, including the containers and the network that connects them.

Create a new folder for your project (e.g., ~/local-ai-stack) and save the following text as a file named docker-compose.yml inside it.

```YAML
version: '3.8'

# This file defines our two-container application:
# 1. ollama-gpu: The AI backend, with GPU access.
# 2. anythingllm: The web interface.

services:
  # Service 1: The Ollama AI Backend (with AMD GPU)
  ollama-gpu:
    image: ollama/ollama:rocm  # Use the official AMD ROCm image
    container_name: ollama-gpu
    networks:
      - llm-net                # Connect to our private network
    devices:
      - /dev/kfd               # Pass the GPU compute device
      - /dev/dri               # Pass the GPU rendering device
    volumes:
      - ollama:/root/.ollama   # Persist models in a named volume
    restart: always

  # Service 2: The AnythingLLM Web Interface
  anythingllm:
    image: mintplexlabs/anythingllm
    container_name: anythingllm
    networks:
      - llm-net                # Connect to the same private network
    ports:
      - "3002:3001"            # Expose the GUI on host port 3002
    volumes:
      - anythingllm:/app/server/storage # Persist chats & documents
    environment:
      - STORAGE_DIR=/app/server/storage # Explicitly set storage path
    depends_on:
      - ollama-gpu             # Wait for Ollama to start first
    restart: always

# Define the private network for our containers
networks:
  llm-net:
    driver: bridge

# Define the persistent storage volumes
volumes:
  ollama:
  anythingllm:
```

## 3. Launching the Application

With the `docker-compose.yml` file saved, launching the entire AI stack is a single command.

1.  **Navigate to your project folder:**
    ```bash
    cd ~/local-ai-stack
    ```

2.  **Start the services:**
    The `-d` flag runs the containers in the background (detached).
    ```bash
    docker-compose up -d
    ```

3.  **Download Your Model:**
    Now, you need to pull your AI model into the `ollama-gpu` container.
    ```bash
    docker exec -it ollama-gpu ollama run codellama:13b
    ```

> **Why `codellama:13b`?**
> During testing, the larger `codellama:34b` model was unstable and caused the Ollama runner to crash (`exit status 2`). The 13B version is far more stable, significantly faster, and still provides enough performance for coding and technical questions.

## 4. Final Configuration

Your AI stack is now running. All that's left is to connect the GUI to the backend.

1.  Open your web browser and navigate to **`http://localhost:3002`**.
    > **Note:** I use port `3002` in this guide because `3001` is a common port for other development services (including Metasploit, which I found in testing). If you need to use a different port, you can easily change it by editing the `ports:` line in the `docker-compose.yml` file.
2.  Follow the AnythingLLM setup wizard.
3.  When you reach **LLM Preference**, select **Ollama**.
4.  In the **Ollama Base URL** field, enter: **`http://ollama-gpu:11434`**
    * (This works because both containers are on the same private Docker network, and ollama-gpu is the container's name from the docker-compose.yml file).
5.  In the **Chat Model** dropdown, you will now see `codellama:13b` available to select.
6.  Save your settings. You can now create a workspace and start chatting with your private, GPU-accelerated AI!

<img width="2554" height="1400" alt="Screenshot From 2025-11-03 12-50-18" src="https://github.com/user-attachments/assets/01bb2e71-edaa-4b0d-a48c-951cb29c4242" />


## 5. Troubleshooting & Key Learnings

This simple setup was the result of many failures. Here are the issues I hit and how this configuration solves them:


 - **Problem:** Installing AMD ROCm drivers directly on the host Debian OS failed with dependency errors.

   - **Solution:** I let the ollama/ollama:rocm container manage all the complex driver dependencies in an isolated environment.

 - **Problem:** The codellama:34b model would crash the Ollama runner (exit status 2).

   - **Solution:** I switched to the smaller, more stable codellama:13b model. The speed and stability are a much better trade-off.

 - **Problem:** AnythingLLM would show "--loading available models--" or Could not resolve host.

   - **Solution:** This was a Docker networking failure. By using Docker Compose, I created a dedicated network (llm-net) that allows the containers to reliably find and communicate with each other by name.

 - **Problem:** Leftover Docker images and volumes from failed attempts were consuming disk space.

   - **Solution:** I used docker system prune -a --volumes to clean up all unused Docker data. I also ran docker exec -it ollama-gpu ollama rm <model_name> to remove old, large models.
