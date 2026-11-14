# OmniRoute & Open WebUI Setup Guide

**OmniRoute** is an open-source, high-performance **AI API gateway and router** that acts as a centralized proxy between your applications (or chat frontends like Open WebUI) and various large language model (LLM) providers such as Google Gemini, Groq, OpenAI, and Anthropic.

### Benefits of OmniRoute
- **Unified API Endpoint:** Switch between different providers using a single OpenAI-compatible API format without changing your underlying application code.
- **Load Balancing & Failover:** Automatically route traffic across multiple providers or keys to ensure high availability and bypass rate limits.
- **Cost Optimization & Routing Rules:** Direct specific prompts or workloads to the most cost-effective or fastest model available.
- **Centralized Key Management:** Keep your provider API keys secure in one gateway rather than scattering them across multiple different client apps.

This guide explains how to run **OmniRoute** (as your AI API gateway and router) and **Open WebUI** (as your frontend chat interface) as separate, clean, standalone Docker containers on an Ubuntu server.

## Prerequisites

* Any Linux server (e.g. Ubuntu 24.04) with **Docker** and **Docker Compose** installed.
* Basic command-line access.

---

## Step 1: Create the Project Directory

Create a dedicated directory on your server to keep your configuration files organized:

```bash
sudo mkdir -p /opt/ai-gateway
cd /opt/ai-gateway
```
---

## Step 2: Configure Environment Variables & Generate Secrets

Create a \`.env\` file for OmniRoute's security keys and configuration:

```bash
sudo nano .env
```

### How to Generate Secure Keys
Before pasting the template, generate random hex strings for your secrets in your terminal using OpenSSL:
* Run `openssl rand -hex 32` for each secret value below to get a unique 64-character token.
* Choose a strong custom password for your `INITIAL_PASSWORD`.

Paste the following variables with your generated keys:

```env
JWT_SECRET=your_generated_jwt_secret_here
INITIAL_PASSWORD=YourSecureAdminPassword123!
API_KEY_SECRET=your_generated_api_key_secret_here
STORAGE_ENCRYPTION_KEY=your_generated_storage_encryption_key_here
STORAGE_ENCRYPTION_KEY_VERSION=v1
MACHINE_ID_SALT=your_generated_machine_id_salt_here
```

---

## Step 3: Create the Docker Compose File

Create the \`docker-compose.yml\` file to run both services on a shared internal network:

```bash
sudo nano docker-compose.yml
```

Paste the following configuration:

```yaml
services:
  omniroute:
    image: diegosouzapw/omniroute:latest
    container_name: omniroute
    restart: unless-stopped
    environment:
      - PORT=20128
      - NODE_ENV=production
      - HOSTNAME=0.0.0.0
      - DATA_DIR=/app/data
      - STORAGE_DRIVER=sqlite
      - APP_LOG_TO_FILE=true
      - JWT_SECRET=\${JWT_SECRET}
      - INITIAL_PASSWORD=\${INITIAL_PASSWORD}
      - API_KEY_SECRET=\${API_KEY_SECRET}
      - STORAGE_ENCRYPTION_KEY=\${STORAGE_ENCRYPTION_KEY}
      - STORAGE_ENCRYPTION_KEY_VERSION=\${STORAGE_ENCRYPTION_KEY_VERSION}
      - MACHINE_ID_SALT=\${MACHINE_ID_SALT}
    ports:
      - "20128:20128"
    volumes:
      - omniroute-data:/app/data

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "3000:8080"
    environment:
      - OPENAI_API_BASE_URL=http://omniroute:20128/v1
    volumes:
      - open-webui-data:/app/backend/data
    depends_on:
      - omniroute

volumes:
  omniroute-data:
  open-webui-data:
```

---

## Step 4: Launch the Containers

Run the following command to download the images and start both containers in the background:

```bash
docker compose up -d
```

Verify that both containers are running properly:

```bash
docker ps
```

---

## Step 5: Generate and Add Provider API Keys (like Gemini & Groq)

Before your models can route through OmniRoute, you need to supply your external provider keys:

1. **Get Google Gemini API Key:**
   - Go to Google AI Studio.
   - Sign in with your Google account and click Get API key.
   - Create or select a Google Cloud project and copy your generated API key.

2. **Get Groq API Key:**
   - Go to the Groq Console.
   - Log in and navigate to the API Keys section.
   - Click Create API Key, give it a name, and copy the generated key.

3. **Add Keys to OmniRoute:**
   - Open your browser and navigate to your OmniRoute dashboard (`http://<your-server-ip>:20128`).
   - Log in using your initial credentials (`INITIAL_PASSWORD`).
   - Navigate to the Providers or API Keys section and add your Gemini and Groq keys so OmniRoute can authenticate requests on your behalf.
  
---

## Step 6: Finalize Setup in Open WebUI

1. Open your browser and navigate to `http://<your-server-ip>:3000`.
2. Create your admin account.
3. Go to **Admin Settings** -> **Connections**.
4. Ensure the OpenAI API Base URL is set to `http://omniroute:20128/v1`.
5. Enter an API key generated from your OmniRoute dashboard, then save. Your models will now appear in the chat interface!
