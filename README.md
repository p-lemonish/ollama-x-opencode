# opencode setup with ollama
Docs: https://opencode.ai/docs/config/

Setup `OLLAMA_DATA` in `.env` to wherever you need to point it to, then using
this repo's `docker-compose.yml` simply run
```bash
docker compose up -d
```

To download a model, such as `qwen3:8b`, and get it ready to run on ollama, run
the following
```bash
docker exec -it ollama ollama pull qwen3:8b

# Or if using docker compose
docker compose exec ollama ollama pull qwen3:8b
```

This following json in `~/.config/opencode/config.json` has worked for me so far
I, like many others have had issues getting the tools working in opencode using
local models.

I found out why agentic actions such as using opencode's provided tools didn't work.
It's because Ollama has set the context window at 4096 for the models.

Even though ollama says the context is larger, for example for qwen3 it
says context is around 40k, upon running the model via ollama, it will use a default
of 4k context, this **has** to be setup by yourself. Also make sure the model
you are planning to use actually supports *agentic* tools.

This was fixed by doing the following:

For example, after pulling `qwen3:8b`
```bash
$ docker compose exec ollama bash
$ ollama run qwen3:8b
>>> /set parameter num_ctx 16384
Set parameter 'num_ctx' to '16384'
>>> /save qwen3:8b-16k
Created new model 'qwen3:8b-16k'
>>> /bye
```

Then using the following `config.json`
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "options": {
        "baseURL": "http://127.0.0.1:11434/v1",
        "apiKey": "ollama"
      },
      "models": {
        "qwen3:8b-16k": {
          "tools": true
        }
      }
    }
  }
}
```

First, to prove the magician's sleeves are empty

```bash
$ ls -la
total 8
drwxrwxr-x 2 lemonish lemonish 4096 Jul  1 17:21 .
drwxrwxr-x 3 lemonish lemonish 4096 Jul  1 17:21 ..
$ cat todo.md
cat: todo.md: No such file or directory
```

Run the command

```bash
$ opencode run "/no_think generate a todo.md file with the contents 'hello world, from qwen3' in it" --model ollama/qwen3:8b-16k

█▀▀█ █▀▀█ █▀▀ █▀▀▄ █▀▀ █▀▀█ █▀▀▄ █▀▀
█░░█ █░░█ █▀▀ █░░█ █░░ █░░█ █░░█ █▀▀
▀▀▀▀ █▀▀▀ ▀▀▀ ▀  ▀ ▀▀▀ ▀▀▀▀ ▀▀▀  ▀▀▀

>  /no_think generate a todo.md file with the contents 'hello world, from qwen3' in it

@  ollama/qwen3:8b-16k

|  Write    ollama/test/todo.md

<think>
</think>

<think>
</think>

</think>
</think>
</think>

Okay, I'll generate a todo.md file with the specified content. Let me create it for you

$ ls
todo.md
$ cat todo.md
hello world, from qwen3
```

ta-da!

## Troubleshooting & Common Hanging Issues on Windows

If your OpenCode integration with Ollama hangs or doesn't get any responses back, this is usually caused by one of the following reasons:

### 1. Localhost resolving to IPv6 (`::1`) instead of IPv4 (`127.0.0.1`)
By default, Ollama on Windows binds only to the IPv4 loopback address (`127.0.0.1:11434`). However, on Windows, `localhost` often resolves to the IPv6 loopback address (`::1`).
*   **The Issue:** When OpenCode attempts to connect to `http://localhost:11434/v1`, it tries the IPv6 address first. If your firewall or system network configuration silently drops these packets instead of immediately rejecting them, the connection will hang until it times out.
*   **The Fix:** Change the `"baseURL"` in your config file to explicitly use `http://127.0.0.1:11434/v1` instead of `localhost`.

### 2. Missing `apiKey` in the OpenAI-compatible configuration
The `@ai-sdk/openai-compatible` provider used in OpenCode expects an API key by default.
*   **The Issue:** If `"apiKey"` is not specified in the configuration and you do not have an `OPENAI_API_KEY` environment variable set, the SDK initialization throws a missing API key error internally. Since the OpenCode UI may not capture or display this promise rejection, it will result in a silent hang.
*   **The Fix:** Add `"apiKey": "ollama"` (or any dummy placeholder string) to the `"options"` object in your `config.json` file.

### 3. WSL2 / Docker Network Bridging
If you are running OpenCode in WSL2 or a Docker container while running Ollama natively on Windows:
*   Make sure you set the environment variable `OLLAMA_HOST` to `0.0.0.0` on Windows so it listens on all interfaces (remember to exit Ollama from the system tray and restart it).
*   Add a Windows Firewall rule to allow port `11434` incoming traffic.
*   Use `http://host.docker.internal:11434/v1` or the Windows host's IP address instead of `localhost` in your config.
