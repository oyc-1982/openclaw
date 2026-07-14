---
summary: "Run OpenClaw with a self-hosted llama.cpp HTTP server"
read_when:
  - You want to connect OpenClaw to a llama.cpp HTTP server
  - You need llama.cpp server setup and configuration guidance
  - You want to use local GGUF models through llama.cpp's OpenAI-compatible API
title: "llama.cpp HTTP Server"
---

OpenClaw can connect to a llama.cpp HTTP server running locally or on your network. The llama.cpp server provides an OpenAI-compatible `/v1` API that OpenClaw can use for chat completions and embeddings.

## Getting started

<Tabs>
  <Tab title="Start llama.cpp server">
    <Steps>
      <Step title="Install llama.cpp">
        Get llama.cpp from [github.com/ggerganov/llama.cpp](https://github.com/ggerganov/llama.cpp):

        ```bash
        git clone https://github.com/ggerganov/llama.cpp
        cd llama.cpp
        make
        ```

        Or use a package manager:

        ```bash
        # Homebrew (macOS)
        brew install llama.cpp

        # Or download pre-built binaries from releases
        ```

      </Step>
      <Step title="Download a model">
        Get a GGUF model file:

        ```bash
        # Example: Download Qwen3.5 3B
        wget https://huggingface.co/Qwen/Qwen3.5-3B-GGUF/resolve/main/qwen3.5-3b-q4_k_m.gguf
        ```

      </Step>
      <Step title="Start the server">
        ```bash
        ./server -m qwen3.5-3b-q4_k_m.gguf --host 127.0.0.1 --port 8080
        ```

        For GPU acceleration:

        ```bash
        # Metal (macOS)
        ./server -m qwen3.5-3b-q4_k_m.gguf --host 127.0.0.1 --port 8080 -ngl 99

        # CUDA (NVIDIA)
        ./server -m qwen3.5-3b-q4_k_m.gguf --host 127.0.0.1 --port 8080 -ngl 99
        ```

      </Step>
      <Step title="Verify the server is running">
        ```bash
        curl http://127.0.0.1:8080/v1/models
        ```

        Should return a JSON response with model information.

      </Step>
    </Steps>

  </Tab>

  <Tab title="Configure OpenClaw">
    <Steps>
      <Step title="Set up provider config">
        Add the llama.cpp provider to your OpenClaw config:

        ```json5
        {
          models: {
            providers: {
              "llama-cpp": {
                baseUrl: "http://127.0.0.1:8080/v1",
                apiKey: "not-needed", // llama.cpp doesn't require auth by default
                api: "openai-completions",
                timeoutSeconds: 300,
                models: [
                  {
                    id: "qwen3.5-3b-q4_k_m.gguf",
                    name: "Qwen3.5 3B (llama.cpp)",
                    reasoning: false,
                    input: ["text"],
                    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                    contextWindow: 32768,
                    maxTokens: 4096,
                  },
                ],
              },
            },
          },
          agents: {
            defaults: {
              model: { primary: "llama-cpp/qwen3.5-3b-q4_k_m.gguf" },
            },
          },
        }
        ```

      </Step>
      <Step title="Select the model">
        ```bash
        openclaw models list --provider llama-cpp
        openclaw models set llama-cpp/qwen3.5-3b-q4_k_m.gguf
        ```

      </Step>
      <Step title="Verify">
        ```bash
        openclaw infer model run \
          --model llama-cpp/qwen3.5-3b-q4_k_m.gguf \
          --prompt "Reply with exactly: pong" \
          --json
        ```

      </Step>
    </Steps>

  </Tab>
</Tabs>

## Configuration reference

### Basic setup

```json5
{
  models: {
    providers: {
      "llama-cpp": {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "not-needed",
        api: "openai-completions",
        timeoutSeconds: 300,
        models: [
          {
            id: "your-model.gguf",
            name: "Your Model Name",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32768,
            maxTokens: 4096,
          },
        ],
      },
    },
  },
}
```

### Key fields

| Field | Required | Description |
|-------|----------|-------------|
| `baseUrl` | yes | llama.cpp server URL with `/v1` path |
| `apiKey` | no | Authentication key (llama.cpp doesn't require one by default) |
| `api` | yes | Must be `"openai-completions"` for llama.cpp's OpenAI-compatible API |
| `timeoutSeconds` | no | Request timeout in seconds (default: 60). Increase for large/slow models |
| `models[].id` | yes | Model identifier (can match the GGUF filename) |
| `models[].contextWindow` | no | Context window size (check model documentation) |
| `models[].maxTokens` | no | Maximum output tokens |

### Multiple models

If your llama.cpp server hosts multiple models (switched via command line or hot-reload), define them all:

```json5
{
  models: {
    providers: {
      "llama-cpp": {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "not-needed",
        api: "openai-completions",
        models: [
          {
            id: "qwen3.5-3b.gguf",
            name: "Qwen3.5 3B",
            input: ["text"],
            contextWindow: 32768,
            maxTokens: 4096,
          },
          {
            id: "llama3.2-3b.gguf",
            name: "Llama 3.2 3B",
            input: ["text"],
            contextWindow: 32768,
            maxTokens: 4096,
          },
        ],
      },
    },
  },
}
```

## Advanced configuration

<AccordionGroup>
  <Accordion title="LAN or remote llama.cpp server">
    For a llama.cpp server on another machine:

    ```json5
    {
      models: {
        providers: {
          "llama-cpp": {
            baseUrl: "http://gpu-server.local:8080/v1",
            apiKey: "not-needed",
            api: "openai-completions",
            timeoutSeconds: 300,
            models: [
              {
                id: "qwen3.5-3b.gguf",
                name: "Qwen3.5 3B",
                input: ["text"],
                contextWindow: 32768,
                maxTokens: 4096,
              },
            ],
          },
        },
      },
    }
    ```

    Make sure the llama.cpp server is bound to the correct interface:

    ```bash
    ./server -m model.gguf --host 0.0.0.0 --port 8080
    ```

    <Warning>
    Only expose llama.cpp to your LAN if you trust all devices on the network. Consider firewall rules or SSH tunneling for additional security.
    </Warning>

  </Accordion>

  <Accordion title="Custom API key">
    If you've configured llama.cpp with authentication (via reverse proxy or custom middleware):

    ```json5
    {
      models: {
        providers: {
          "llama-cpp": {
            baseUrl: "http://127.0.0.1:8080/v1",
            apiKey: "your-api-key",
            api: "openai-completions",
            models: [...],
          },
        },
      },
    }
    ```

    Or use an environment variable:

    ```bash
    export LLAMA_CPP_API_KEY="your-api-key"
    ```

    ```json5
    {
      models: {
        providers: {
          "llama-cpp": {
            baseUrl: "http://127.0.0.1:8080/v1",
            apiKey: "LLAMA_CPP_API_KEY",
            api: "openai-completions",
            models: [...],
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Reasoning models">
    For reasoning-capable models (like R1 variants):

    ```json5
    {
      models: {
        providers: {
          "llama-cpp": {
            baseUrl: "http://127.0.0.1:8080/v1",
            apiKey: "not-needed",
            api: "openai-completions",
            models: [
              {
                id: "r1-3b.gguf",
                name: "R1 3B",
                reasoning: true, // Enable reasoning mode
                input: ["text"],
                contextWindow: 32768,
                maxTokens: 4096,
              },
            ],
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Vision models">
    For vision-capable models:

    ```json5
    {
      models: {
        providers: {
          "llama-cpp": {
            baseUrl: "http://127.0.0.1:8080/v1",
            apiKey: "not-needed",
            api: "openai-completions",
            models: [
              {
                id: "llava-1.5-7b.gguf",
                name: "LLaVA 1.5 7B",
                reasoning: false,
                input: ["text", "image"], // Mark as vision-capable
                contextWindow: 4096,
                maxTokens: 512,
              },
            ],
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Local service management">
    OpenClaw can automatically start and stop the llama.cpp server on demand using `localService`:

    ```json5
    {
      models: {
        providers: {
          "llama-cpp": {
            baseUrl: "http://127.0.0.1:8080/v1",
            apiKey: "not-needed",
            api: "openai-completions",
            timeoutSeconds: 300,
            localService: {
              command: "/absolute/path/to/llama.cpp/server",
              args: [
                "-m", "/absolute/path/to/model.gguf",
                "--host", "127.0.0.1",
                "--port", "8080",
                "-ngl", "99", // GPU layers
              ],
              cwd: "/absolute/path/to/llama.cpp",
              healthUrl: "http://127.0.0.1:8080/v1/models",
              readyTimeoutMs: 180000, // 3 minutes to start
              idleStopMs: 300000, // Stop after 5 minutes idle (0 = never)
            },
            models: [
              {
                id: "model.gguf",
                name: "My Model",
                input: ["text"],
                contextWindow: 32768,
                maxTokens: 4096,
              },
            ],
          },
        },
      },
    }
    ```

    See [Local model services](/gateway/local-model-services) for full details.

  </Accordion>

  <Accordion title="Context window tuning">
    Adjust context window based on your hardware and model:

    ```json5
    {
      models: {
        providers: {
          "llama-cpp": {
            baseUrl: "http://127.0.0.1:8080/v1",
            apiKey: "not-needed",
            api: "openai-completions",
            contextWindow: 16384, // Provider-level default
            models: [
              {
                id: "small-model.gguf",
                name: "Small Model",
                contextWindow: 8192, // Override for this model
                maxTokens: 2048,
              },
              {
                id: "large-model.gguf",
                name: "Large Model",
                contextWindow: 32768,
                maxTokens: 4096,
              },
            ],
          },
        },
      },
    }
    ```

    llama.cpp server context is controlled by the `--ctx-size` flag when starting the server.

  </Accordion>
</AccordionGroup>

## Troubleshooting

<AccordionGroup>
  <Accordion title="Connection refused">
    Check if the llama.cpp server is running:

    ```bash
    curl http://127.0.0.1:8080/v1/models
    ```

    If it fails, start the server:

    ```bash
    ./server -m model.gguf --host 127.0.0.1 --port 8080
    ```

  </Accordion>

  <Accordion title="Model not found">
    Verify the model ID matches what llama.cpp reports:

    ```bash
    curl http://127.0.0.1:8080/v1/models | jq
    ```

    Update your config to use the exact model ID from the response.

  </Accordion>

  <Accordion title="Request timeout">
    Large models or slow hardware may need longer timeouts:

    ```json5
    {
      models: {
        providers: {
          "llama-cpp": {
            timeoutSeconds: 600, // 10 minutes
            // ... rest of config
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Out of memory">
    If llama.cpp crashes or returns errors about memory:

    - Reduce `--ctx-size` when starting the server
    - Use fewer GPU layers (`-ngl`)
    - Choose a smaller quantization (e.g., Q4_K_M instead of Q8_0)
    - Close other applications using GPU/VRAM

  </Accordion>

  <Accordion title="Tool calling fails">
    llama.cpp's tool calling support varies by model and version. If tools fail:

    - Ensure you're using a recent llama.cpp build
    - Try a model known to support tool calling (e.g., Hermes, Mistral)
    - Set `compat.supportsTools: false` on the model entry as a workaround

    ```json5
    {
      models: [
        {
          id: "model.gguf",
          name: "Model",
          compat: { supportsTools: false },
        },
      ],
    }
    ```

  </Accordion>

  <Accordion title="Proxy header warnings">
    If you see warnings about "untrusted proxy headers" or similar:

    This typically happens when running behind a reverse proxy. For local llama.cpp connections, ensure:

    - `baseUrl` points directly to the llama.cpp server (not through a proxy)
    - No intermediate proxy is adding/modifying headers
    - If using a proxy, configure `gateway.trustedProxies` appropriately

    See [Trusted proxy auth](/gateway/trusted-proxy-auth) for proxy setup guidance.

  </Accordion>

  <Accordion title="Skill path escape warnings">
    Warnings about skill paths escaping usually indicate a configuration issue:

    - Check that plugin paths are correctly specified
    - Ensure no relative paths escape the intended directory
    - Run `openclaw doctor` to check for configuration issues

  </Accordion>
</AccordionGroup>

## Performance tips

<AccordionGroup>
  <Accordion title="GPU acceleration">
    For best performance, use GPU offloading:

    ```bash
    # macOS (Metal)
    ./server -m model.gguf --host 127.0.0.1 --port 8080 -ngl 99

    # Linux/Windows (CUDA)
    ./server -m model.gguf --host 127.0.0.1 --port 8080 -ngl 99

    # Vulkan (AMD/Intel)
    GGML_VK_DEVICE=0 ./server -m model.gguf --host 127.0.0.1 --port 8080 -ngl 99
    ```

    Adjust `-ngl` based on your VRAM. Start high and reduce if you get OOM errors.

  </Accordion>

  <Accordion title="Batching and concurrency">
    llama.cpp server supports batching multiple requests. For better throughput:

    - Use `--batch-size 512` (or higher if VRAM allows)
    - Set `--ubatch-size 512` for uniform batch size
    - Consider `--sequence-parallel` for very large contexts

  </Accordion>

  <Accordion title="Memory mapping">
    For faster model loading:

    ```bash
    ./server -m model.gguf --mlock --host 127.0.0.1 --port 8080
    ```

    `--mlock` keeps the model in RAM, preventing swap thrashing.

  </Accordion>

  <Accordion title="Keep model loaded">
    By default, llama.cpp may unload the model between requests. To keep it loaded:

    ```bash
    ./server -m model.gguf --keep-alive 3600 --host 127.0.0.1 --port 8080
    ```

    Or set `idleStopMs: 0` in the `localService` config to prevent OpenClaw from stopping it.

  </Accordion>
</AccordionGroup>

## Related

<CardGroup cols={2}>
  <Card title="Local model services" href="/gateway/local-model-services" icon="server">
    Automatic start/stop of local model servers.
  </Card>
  <Card title="Self-hosted providers" href="/concepts/model-providers#self-hosted" icon="cloud">
    Self-hosted provider setup patterns.
  </Card>
  <Card title="llama.cpp Provider (embeddings)" href="/plugins/reference/llama-cpp" icon="database">
    Local GGUF embeddings through node-llama-cpp.
  </Card>
  <Card title="Configuration reference" href="/gateway/configuration-reference" icon="gear">
    Full OpenClaw configuration reference.
  </Card>
</CardGroup>
