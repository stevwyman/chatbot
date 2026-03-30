# Chatbot Recipes

just a collection of recipes for local chatbots

## simple LLAMA-CPP

using:
- granite-4.0-micro-Q4_K_M.gguf

MODEL=/models/granite-4.0-h-micro-Q4_K_M.gguf

podman compose up -d

## OLLAMA

FOr use with a MAC Air using M-architecture

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
