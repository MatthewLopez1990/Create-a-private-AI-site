# Your Personal AI Setup: Open Web UI with Searxng and Caddy

## Introduction: What is This All About?

Welcome to the documentation for your personal AI setup! This guide will walk you through all the pieces of the system we've just configured. The goal is to make everything as clear as possible, even if you're new to this kind of technology.

Think of this as building your own private, smart assistant, like ChatGPT or Google Assistant, but one that you control and that respects your privacy.

This setup has a few moving parts, and this document will explain each one, what it does, and how they all work together.

**Note on Hardware**: If you do not already have a server, a **Raspberry Pi 5** can be purchased for around **$150**, which can act as a powerful and efficient server gateway for this entire setup.

## The Building Blocks: What Are the Components?

Our setup is made of four main components. Let's get to know them.

### 1. Open Web UI: Your Gateway to AI

-   **What it is**: Open Web UI is a website that you can use to talk to "Large Language Models" (LLMs). An LLM is a powerful AI, like the one you're talking to right now, that can understand and generate human-like text.
-   **Why it's great**: It gives you a clean, user-friendly chat interface, similar to what you might find on other AI websites. You can have multiple conversations, manage different AI models, and customize the experience.
-   **In our setup**: We run Open Web UI in a "container". A container is like a mini, self-contained computer that runs just one application. This keeps it tidy and separate from the rest of your system.

### 2. Searxng: Your Private Search Engine

-   **What it is**: Searxng is a special kind of search engine called a "metasearch engine". This means it doesn't have its own search index, but instead, it gets results from many other search engines (like Google, Bing, etc.).
-   **Why it's great for privacy**: Searxng acts as a middleman. When you search, Searxng gets the results for you, so the big search engines never know it was you who searched. This is a huge win for privacy. The version we are using is also configured with privacy in mind.
-   **In our setup**: Open Web UI uses Searxng to give your AI "eyes" on the internet. When you ask the AI a question about a recent event, it can use Searxng to look up the answer. Like Open Web UI, Searxng also runs in its own container.

### 3. Caddy: The Friendly Doorman for Your Website

-   **What it is**: Caddy is a modern "web server" and "reverse proxy". A web server is a program that sends web pages to your browser. A reverse proxy is like a receptionist for your web services. It takes all incoming requests and directs them to the right place.
-   **Why it's amazing**: Caddy is famous for being easy to set up. It also automatically handles "HTTPS" for you. HTTPS is the technology that provides a secure, encrypted connection to a website (you see a padlock icon in your browser's address bar). Caddy will get and renew the security certificates for you, which can be a complex task with other web servers.
-   **In our setup**: Caddy is the front door to your Open Web UI. When you go to `https://yourdomain.duckdns.org`, you are talking to Caddy. Caddy then forwards your request to the Open Web UI container in a secure and efficient way.

### 4. Groq/Fireworks.ai: The Privacy-First AI Model

-   **What it is**: Groq is the AI model provider that powers the intelligence of this system. While Open Web UI provides the interface, Groq provides the actual "thinking" and text generation.
-   **Why it's essential**: We have specifically chosen Groq as the default model provider due to its **high standard of privacy**. Unlike many other AI services, Groq and Fireworks.ai ensure that your data is not used for model training, keeping your conversations private and secure. It is also incredibly fast, providing a snappy and responsive user experience.
-   **In our setup**: You will connect your Open Web UI instance to Groq using a secure API key. This allows you to leverage state-of-the-art AI capabilities while maintaining strict control over your personal data.

## The Blueprint: How It's All Configured

Now let's look at the configuration files that tell our components how to behave.

### `docker-compose.yml`: The Instruction Manual for Our Containers

Docker is a tool for running applications in containers. Docker Compose is a tool for defining and running multi-container Docker applications. The `docker-compose.yml` file is the instruction manual that tells Docker Compose how to set up our Open Web UI and Searxng containers.

Here's a breakdown of the file:

```yaml
version: "3.8" # Specifies the version of the Docker Compose file format.

services: # This is where we define the containers we want to run.
  open-webui: # The name of our first service.
    image: ghcr.io/open-webui/open-webui:main # The container image to use. An image is like a blueprint for a container.
    container_name: open-webui # A custom name for our container.
    ports:
      - "3000:8080" # This maps port 3000 on your computer to port 8080 inside the container. So when you access port 3000, you are talking to the Open Web UI application.
    environment: # These are like settings for the Open Web UI application.
      - ENABLE_RAG_WEB_SEARCH=True # Enables the web search feature.
      - RAG_WEB_SEARCH_ENGINE=searxng # Tells Open Web UI to use Searxng for web searches.
      - RAG_WEB_SEARCH_RESULT_COUNT=3 # The number of search results to use.
      - SEARXNG_QUERY_URL=http://searxng:8080/search?q=<query> # The address of the Searxng service.
    networks:
      - webui-network # Connects this container to a network named "webui-network".
    depends_on:
      - searxng # Tells Docker to start the "searxng" container before starting this one.

  searxng: # The name of our second service.
    image: searxng/searxng:latest # The container image for Searxng.
    container_name: searxng # A custom name for our Searxng container.
    ports:
      - "8888:8080" # Maps port 8888 on your computer to port 8080 inside the Searxng container.
    volumes:
      - ./searxng_data/config:/etc/searxng:rw # This syncs the "./searxng_data/config" folder on your computer with the "/etc/searxng" folder inside the container. This is how you can customize Searxng's settings.
    networks:
      - webui-network # Connects this container to the same network as Open Web UI.

networks: # This is where we define our networks.
  webui-network: # The name of our network.
    driver: bridge # The "bridge" driver is the standard Docker network driver.
```

### `Caddyfile`: Caddy's Simple Instructions

The `Caddyfile` is the configuration file for Caddy. It's designed to be very simple and human-readable. Ours is located at `/etc/caddy/Caddyfile`.

```caddy
yourdomain.duckdns.org, YOUR_PUBLIC_IP { # This tells Caddy to listen for requests for your domain name and IP address.
    reverse_proxy 127.0.0.1:3000 # This is the core instruction. "reverse_proxy" tells Caddy to forward the request to another service. "127.0.0.1:3000" is the address of your Open Web UI container (from Caddy's perspective, it's on the same machine at port 3000).
}
```

### Configuring Open Web UI with Your AI Model: The Groq Advantage

To make Open Web UI useful, you need to connect it to an AI model provider. **We strongly recommend using Groq for AI inference due to its exceptional privacy features.**

#### Why Groq is the Best Choice for Privacy

Groq stands out from other AI providers because of its commitment to user privacy:

-   **No Training on Your Data**: Unlike many AI providers, **Groq does NOT train their models on your conversations**. Your inputs and outputs remain yours alone.
-   **Configurable Privacy Settings**: You can configure Groq to ensure that **none of your inputs or outputs are monitored or stored** by their systems. This means your conversations are truly private.
-   **Complete Data Control**: When combined with self-hosted Open Web UI and Searxng, you have full control over your AI experience. Your data never leaves your control unless you explicitly choose to send it to Groq for inference.
-   **Fast Inference**: Groq is known for extremely fast inference speeds, giving you a responsive AI experience without sacrificing privacy.

This combination of privacy, speed, and power makes Groq the ideal choice for anyone who values their data security and wants to maintain control over their AI interactions.

#### Setting Up Groq with Open Web UI

1.  **Get your API Key**:
    *   Go to the  example: [Groq website](https://console.groq.com) and sign up for an account if you don't have one.
    *   Navigate to the "API Keys" section in your account dashboard.
    *   Create a new API key.
    *   **Important**: An API Key is like a password for a service. **Treat it like a password!** Do not share it, do not post it publicly, and do not save it in any public files.

2.  **Configure Privacy Settings (Highly Recommended)**:
    *   In your Groq account settings, look for privacy or data retention options.
    *   **Disable any data logging or monitoring features** to ensure your inputs and outputs are not stored.
    *   This step is crucial for maintaining maximum privacy - take the time to review all available privacy settings.

3.  **Configure Open Web UI**:
    *   Open your web browser and go to `https://yourdomain.duckdns.org` (replace with your actual domain).
    *   You should see the Open Web UI interface.
    *   Click on the "Settings" or "Admin" area. It might be behind a gear icon or your user profile.
    *   Look for a section related to "Model Providers", "Connections", or "API Settings".
    *   Add a new model provider with the following information:
        *   **Model Provider URL** (or similar): `https://api.groq.com/openai/v1`, or `https://api.fireworks.ai/inference/v1`.
        *   **API Key**: Copy and paste the API key you got from Groq and or Fireworks.ai.
    *   Save your settings.

Now you should be able to start a new chat and select one of the Groq models to talk to! You'll enjoy fast, private AI inference without worrying about your data being used for training or stored indefinitely.

## The Network: How the Pieces Connect

### The Private Docker Network

The `open-webui` and `searxng` containers are connected to a private virtual network called `webui-network`. Think of this as a private phone line between the two services. They can talk to each other using their container names as if they were domain names. This is why the `SEARXNG_QUERY_URL` in the `docker-compose.yml` file can be `http://searxng:8080/search?q=<query>`. This is more secure and efficient than exposing both services to your main home network.

### Port Forwarding: Opening the Door to the Internet

"Port forwarding" is a setting on your home router that directs incoming traffic from the internet to a specific device on your local network.

Imagine your router is a receptionist in an office building. When a visitor arrives (an internet request), they ask for a specific person (a port number). Port forwarding is like giving the receptionist a list of instructions, like "If someone asks for person #80 or #443, send them to your computer's office".

In our case:
-   **Ports 80 and 443** are the standard ports for web traffic (HTTP and HTTPS).
-   Your Caddy server is listening on these ports for incoming requests.
-   You need to tell your router to forward all traffic on ports 80 and 443 to the local IP address of the computer running Caddy (e.g., **192.168.1.100** - find your actual local IP using `ip addr` or `ipconfig`).

The process for setting up port forwarding is different for every router. You will need to log in to your router's administration page and look for a section called "Port Forwarding", "Virtual Servers", or something similar.

## Step-by-Step: How We Built This

Here is a summary of the steps that were taken to create this setup. You can use this as a reference if you ever need to set it up again.

1.  **Install Docker and Docker Compose**:
    *   These tools are essential for running our containerized applications.
    *   The installation process varies depending on your operating system. You can find the official instructions at the [Docker website](https://docs.docker.com/get-docker/).

2.  **Create the `docker-compose.yml` file**:
    *   A file named `docker-compose.yml` was created in your home directory with the content described in the "Blueprint" section.

3.  **Install Caddy**:
    *   Caddy was installed using the system's package manager. The following commands were used:
        ```bash
        # Install dependencies
        sudo apt update && sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl gnupg
        # Add the Caddy GPG key
        curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
        # Add the Caddy repository
        curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
        # Update package list and install Caddy
        sudo apt update
        sudo apt install caddy -y
        ```

4.  **Configure Caddy**:
    *   A `Caddyfile` was created at `/etc/caddy/Caddyfile` with the reverse proxy configuration.

5.  **Start the Services**:
    *   The Open Web UI and Searxng services are started with the command:
        ```bash
        docker-compose up -d
        ```
        (The `-d` flag means "detached mode", which runs the containers in the background).
    *   The Caddy service is managed by the system and was started automatically after installation. If you need to restart it, you can use the command:
        ```bash
        sudo systemctl restart caddy
        ```

6.  **Configure Port Forwarding**:
    *   As described in the "Networking" section, ports 80 and 443 need to be forwarded to the local IP address of your server in your router's settings.

And that's it! We hope this detailed guide helps you understand and enjoy your new private AI setup.
