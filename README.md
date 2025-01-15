# Local AI with Ollama and OpenWebUI
This repository allows you to **run AI models locally** on your machines.

Under the hood, [OpenWebUI](https://docs.openwebui.com/) is used to manage the AI models and [ollama](https://ollama.com/) is used to actually run them.

Optionally, we can serve them behind a **traefik** reverse-proxy and apply domains to each service and also enable GPU support.

This also works on WSL!

## Getting started
First, you need to install [`Docker CE`](https://docs.docker.com/engine/install) and [`docker-compose-plugin`](https://docs.docker.com/compose/install/linux/#install-using-the-repository).  
I have covered this in my WSL setup guide: [install docker](https://github.com/Iyashi/WSL#install-docker)

Next, clone this repository and set it up.
```bash
git clone git@github.com:Iyashi/local-ai.git
cd local-ai
cp example.env .env
```

Now start the stack:
```bash
docker compose up -d
docker compose logs -f
```

You should be able to access the OpenWebUI web interface at http://localhost:11444 now.

However, you don't have any models yet.

## Download models
You can browse through the [ollama models database](https://ollama.com/search) and pick what you need.

Personally, for my development environment, I use
- `nomic-embed-text` for embedding
- `deepseek-coder-v2:16b` for chatting in VSCode (**ATTENTION!** Requires ~12gb RAM/VRAM)
- `qwen2.5-coder:7b` for code completion
- `llama3.1:8b` for various other stuff (like document analysis)

To pull a model, simply run `docker compose exec ollama ollama pull modelname[:version]`.

Once you've downloaded some models, you can already chat with them in OpenWebUI by selecting them from the dropdown menu.

## Use for development (VSCode)
I use the VSCode extension `Continue.continue`. But there are many more extensions to choose from.

`Continue.continue` adds chat and code completion and other capabilities. It's a great extension to leverage AI for development without leaving your editor.

Once installed, you should open the `Continue` settings by pressing `Ctrl+Shift+P` and then enter `Continue: Open Continue config`.  
Add your downloaded models and remember to set the correct `apiBase` url when using a proxy or to replace port `11434` with your `OLLAMA_PORT` when using direct port mappings:
```json
{
  "models": [
    {
      // ATTENTION: Requires ~12gb RAM/VRAM
      "title": "DeepSeek Coder v2 [16B]",
      "provider": "ollama",
      "model": "deepseek-coder-v2:16b",
      "apiBase": "http://api.ollama.local/" // when using proxy and domain
      // "apiBase": "http://localhost:11434/" // when using direct port mappings
    },
    {
      "title": "Qwen 2.5 Coder [7B]",
      "model": "qwen2.5-coder:7b",
      "provider": "ollama",
      "apiBase": "http://api.ollama.local/" // when using proxy and domain
      // "apiBase": "http://localhost:11434/" // when using direct port mappings
    },
    {
      "title": "Autodetect",
      "model": "AUTODETECT",
      "provider": "ollama",
      "apiBase": "http://api.ollama.local/" // when using proxy and domain
      // "apiBase": "http://localhost:11434/" // when using direct port mappings
    }
  ],
  "tabAutocompleteModel": {
    "title": "AutoComplete",
    "provider": "ollama",
    "model": "qwen2.5-coder:7b",
    "apiBase": "http://api.ollama.local/" // when using proxy and domain
    // "apiBase": "http://localhost:11434/" // when using direct port mappings
  },
  "embeddingsProvider": {
    "title": "Embeddings",
    "provider": "ollama",
    "model": "nomic-embed-text",
    "apiBase": "http://api.ollama.local/" // when using proxy and domain
    // "apiBase": "http://localhost:11434/" // when using direct port mappings
  },
  "customCommands": [
    // ...
  ],
  "contextProviders": [
    // ...
  ],
  "slashCommands": [
    // ...
  ]
}
```

> **Notice**: You can also add other APIs like ChatGPT to Continue. Refer to their documentation.

Now you should have auto code completion in VSCode and should be able to start a new chat session with `Ctrl+Shift+P` `> Continue: New Session`.

## Serve behind a proxy with custom domains
First, you need to configure the domains in the `.env` file. (`*_DOMAIN`)

To be able to resolve them locally, you need to add the domains to your `/etc/hosts` file (`C:\Windows\system32\drivers\etc\hosts` on Windows).  
For example:
```hosts
127.0.0.1  ollama.local api.ollama.local traefik.ollama.local
```
> - `ollama.local` (env `WEBUI_DOMAIN`) is for the OpenWebUI service.
> - `api.ollama.local` (env `OLLAMA_DOMAIN`) is for the Ollama service.
> - `traefik.ollama.local` (env `TRAEFIK_DOMAIN`) is for the Traefik Dashboard service.

Lastly, you need add `:compose.proxy.yml` at the end of `COMPOSE_FILE` line in the `.env` file.  
For example:
```env
COMPOSE_FILE=compose.yml:compose.proxy.yml:compose.gpu.yml
```

Restart the stack using `docker compose down && docker compose up -d` and you're done!

### Serve the proxy behind another proxy
If you already have an external proxy which blocks port 80 and/or 443, you can use it to direct all traffic for `WEBUI_DOMAIN`, `OLLAMA_DOMAIN` and `TRAEFIK_DOMAIN` to the internal ollama-proxy.

Here is an example for an external traefik proxy with docker provider enabled:

Just add a `compose.override.yml` with the following content and replace the `traefik-extern` with your own external proxy network:
```yml
networks:
  traefik-extern:
    external: true

services:
  proxy:
    networks:
      - traefik-extern
    ports: !reset []
    # Tell the system-wide traefik to proxy this container
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=traefik-extern"
      - "traefik.http.routers.ollama.rule=Host(`${WEBUI_DOMAIN}`) || Host(`${OLLAMA_DOMAIN}`) || Host(`${TRAEFIK_DOMAIN}`)"
```

Lastly, add `:compose.override.yml` to `COMPOSE_FILE` env var in the `.env` file. It must be the last entry in the list!

## Enable GPU support
> **Notice**: I only own an NVIDIA GPU, so I can only provide instructions for that. For an AMD or Intel GPU, you should be able to follow the same steps, but with different drivers and tools.

To enable GPU acceleration, you need to install the [`nvidia-container-toolkit`](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html).  

I have already covered this in my [WSL setup guide](https://github.com/Iyashi/WSL) and it also works on native linux, so you can just follow the steps here: [install nvidia-container-toolkit](https://github.com/Iyashi/WSL#install-nvidia-container-toolkit-for-docker-gpu-integration).

To test if the GPU passthrough is working, you can run the following command:
```bash
docker run --rm -it --gpus=all nvcr.io/nvidia/k8s/cuda-sample:nbody nbody -gpu -benchmark
```
> **Notice**: A reboot may be required after installing drivers.

Next, you have to configure the ollama service to use the GPU.  
Just add `:compose.gpu.yml` to `COMPOSE_FILE` env var in the `.env` file and configure your driver and devices accordingly.  
For example:
```env
COMPOSE_FILE=compose.yml:compose.ports.yml:compose.gpu.yml

OLLAMA_GPU_DRIVER=nvidia # or "amd", "intel", etc. if you setup the driver correctly
OLLAMA_GPU_DEVICES=all   # or a list of specific device IDs like "0,1"
```

Restart the stack using `docker compose down && docker compose up -d` and you're done!
