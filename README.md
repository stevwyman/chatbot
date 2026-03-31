# Chatbot Recipes

just a collection of recipes for local chatbots

note: I am using a MacBook with Mx, hence some of the decisions below are based on that.

## simple LLAMA-CPP

not recommended for Mac Silicone

using:
- granite-4.0-micro-Q4_K_M.gguf

```yaml
services:
  llm:
    image: ghcr.io/abetlen/llama-cpp-python:latest
    container_name: llama-server
    ports:
      - "11223:8000"
    volumes:
      - ./models:/models
    environment:
      - MODEL=/models/granite-4.0-micro-Q4_K_M.gguf
    command: >
      python3 -m llama_cpp.server
      --model /models/granite-4.0-micro-Q4_K_M.gguf
      --host 0.0.0.0
      --port 8000
    restart: unless-stopped

  webui:
    image: ghcr.io/open-webui/open-webui:latest
    container_name: open-webui
    ports:
      - "3000:8080"
    environment:
      - OPENAI_API_BASE_URL=http://llm:8000/v1
      # For Podman on Mac, this helps resolve the internal container network
      - DOCKER_HOST=unix:///var/run/docker.sock
      - OPENAI_API_KEY=local
    depends_on:
      - llm
    restart: unless-stopped
````

## OLLAMA

For use with a MAC Air and Apple Silicone

compose:

```yaml
services:
  # The Inference Engine (Llama Stack / Ollama)
  # Note: Ollama is the most stable 'stack' for Mac Podman users
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    volumes:
      - ollama_data:/root/.ollama
    ports:
      - "11434:11434"
    restart: unless-stopped

  # The Frontend
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    dependencies:
      - ollama
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      # For Podman on Mac, this helps resolve the internal container network
      - DOCKER_HOST=unix:///var/run/docker.sock
    volumes:
      - open-webui_data:/app/backend/data
    restart: unless-stopped

volumes:
  ollama_data:
  open-webui_data:
````

To get the Ollama service to "see" and use a specific Granite model file (like a .gguf) from your /models folder, you have to create a Modelfile. Ollama doesn't automatically scan directories for models; it needs this "recipe" to know how to handle the weights.

Here is the step-by-step to bridge that gap.

```sh
podman exec -it ollama ollama create local-granite -f /models/granite.Modelfile
````

You can also download odels from ollame site. Here you can use the granite3.1-dense

## Ollama with external Search

```yaml
services:
  # The All-in-One Container: Open WebUI + Ollama
  open-webui:
    image: ghcr.io/open-webui/open-webui:ollama
    container_name: open-webui
    ports:
      - "3000:8080"
    volumes:
      - open-webui_data:/app/backend/data
      - ollama_data:/root/.ollama
    environment:
      # Enable external real-time data via web search
      - ENABLE_RAG_WEB_SEARCH=True
      - RAG_WEB_SEARCH_ENGINE=searxng
      # The <query> tag is explicitly required by Open WebUI
      - SEARXNG_QUERY_URL=http://searxng:8080/search?q=<query>
    depends_on:
      - searxng
    restart: unless-stopped

  # The External Data Source: SearXNG (Real-time Web Search)
  searxng:
    image: searxng/searxng:latest
    container_name: searxng
    ports:
      - "8080:8080"
    volumes:
      # We mount a local folder to hold the SearXNG configuration
      - ./searxng:/etc/searxng:rw
    restart: unless-stopped

volumes:
  open-webui_data:
  ollama_data:
````

Essential Configuration Step

For Open WebUI to understand the data coming from SearXNG, SearXNG must be told to output its data in JSON format rather than standard HTML.

Create a folder named searxng in the same directory as your compose.yaml.

Inside that folder, create a file named settings.yml.

Paste the following minimal configuration into settings.yml:

```yaml
search:
  formats:
    - html
    - json # This is required for Open WebUI to read the data
````
