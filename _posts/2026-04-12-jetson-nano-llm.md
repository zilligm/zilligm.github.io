---
layout: posts
title: "Running Gemma 4 on a NVIDIA Jetson Orin Nano"
date: 2026-04-12
published: true
description: "How I set up the Gemma 4 E4B model on a NVIDIA Jetson Orin Nano to power local LLM applications and what I learned about performance on 8GB of RAM."
---

Over the last couple of years, I've been tinkering with LLMs and AI agents. From the start, I wanted to keep everything local — mostly for privacy, but also to avoid the risk of unexpected cloud bills if something in an autonomous agent loop goes sideways. So I've been running models locally through [Ollama](https://ollama.com/) on my desktop.

That setup works, but it comes with an obvious limitation: my PC has to stay on. If I want an agent to run overnight or while I'm away, the machine needs to be awake and dedicating its GPU to inference. That's not great for power consumption, and it locks me out of using the GPU for anything else.

When Google released [Gemma 4](https://blog.google/technology/developers/gemma-4-model-family/), I was immediately interested. I had been running Gemma 3 12B through Ollama and was generally happy with its performance, but 12B parameters is too large for anything other than a full desktop GPU. Gemma 4's E4B variant — a mixture-of-experts model that activates only 4 billion parameters per forward pass — changed the equation. Small enough to fit in 8GB of RAM, but punching well above its weight class in benchmarks.

I had a [NVIDIA Jetson Orin Nano](https://www.nvidia.com/en-us/autonomous-machines/embedded-systems/jetson-orin/) sitting in a drawer. I'd used it a while back for my [3D pose reconstruction project](/projects/3d_pose_reconstruction.html), but it had been collecting dust since. The Jetson Orin Nano is a compact edge computing board with a 1024-core NVIDIA GPU and 8GB of shared memory — originally designed for robotics and edge AI workloads. If Gemma 4 E4B could fit, I'd have a dedicated, always-on LLM server drawing a fraction of the power my desktop uses.

So I decided to give it a try.

<!-- image: photo of the Jetson Orin Nano setup (board, SSD, cables) -->

## What You Need

1. NVIDIA Jetson Orin Nano Developer Kit
2. A host PC (to run the NVIDIA SDK Manager)
3. USB-C cable
4. An SSD for the Jetson — strongly recommended, since LLM model files can easily exceed what fits on an SD card
5. (Optional) Monitor, mouse, and keyboard for initial setup


## Setting Up the Jetson

The Jetson runs a version of Ubuntu, but it's not a standard PC. You can't just flash a generic Ubuntu image and expect the GPU to work. The board requires NVIDIA's [JetPack SDK](https://developer.nvidia.com/embedded/jetpack), which provides the full software stack: CUDA, cuDNN, TensorRT, and the specialized drivers for the Jetson's ARM + GPU architecture. Without it, you have a Linux box with no hardware acceleration.

NVIDIA's [SDK Manager](https://docs.nvidia.com/sdk-manager/install-with-sdkm-jetson/index.html) handles the flashing process. It installs the OS and the entire JetPack stack onto the Jetson over USB from a host PC.

I already had my Jetson set up from the previous project, but it was running JetPack r36.4. Running Gemma 4 on the Jetson [requires r36.5](https://forums.developer.nvidia.com/t/no-luck-with-gemma-4-on-jetson-nano-super/365620/). I tried upgrading in place, but ran into issues, so I ended up doing a clean flash with the SDK Manager.

A small aside on the SDK Manager experience: until recently, NVIDIA only supported running it on Ubuntu, and it required a specific older release. Last time I set up this Jetson, I had to edit a file inside the SDK Manager installation to spoof my Ubuntu version — pretending to be the older release it expected. Hacky, but it worked. This time around, they support Windows, which made the whole process much smoother.


## Running the LLM

Once JetPack r36.5 is installed, you're technically ready to serve an LLM. NVIDIA provides a pre-built [llama.cpp Docker image](https://github.com/NVIDIA-AI-IOT/llama_cpp_jetson) optimized for Jetson, with CUDA support baked in. The minimal command to get Gemma 4 E4B running is:

```bash
sudo docker run -it --rm --pull always \
  --runtime=nvidia --network host \
  -v $HOME/.cache/huggingface:/root/.cache/huggingface \
  ghcr.io/nvidia-ai-iot/llama_cpp:gemma4-jetson-orin \
  llama-server -hf ggml-org/gemma-4-E4B-it-GGUF:Q4_K_M
```

This pulls the Q4_K_M quantized model from Hugging Face and starts llama-server with a web interface. Before running it, make sure to set the Jetson's power mode to **MAXN SUPER** (via `nvpmodel`) to unlock the full clock speeds.

That one-liner works, but with only 8GB of shared memory (CPU and GPU share the same pool), the Jetson starts struggling as soon as the context window grows. I found that combining Docker memory limits with llama-server tuning flags keeps things stable.


## Memory Optimization

First, I set up a swap file. The Jetson's default ZRAM swap is too small for LLM workloads, so I replaced it with a 16GB file-backed swap:

1. Disable the default ZRAM (it can conflict with larger allocations):

   ```bash
   sudo systemctl disable nvzramconfig
   ```

2. Allocate a 16GB swap file:

   ```bash
   sudo fallocate -l 16G /swapfile
   ```

3. Secure and format it:

   ```bash
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   ```

4. Enable it:

   ```bash
   sudo swapon /swapfile
   ```

5. Make it persist across reboots:

   ```bash
   echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
   ```

Since I plan the Jetson purely as a headless LLM server — no monitor, no desktop environment — I also disabled the GUI to reclaim a bit more memory:

```bash
sudo systemctl set-default multi-user.target
```

(You can always bring it back with `sudo systemctl set-default graphical.target`.)

With swap in place and the GUI off, I run the container with explicit memory constraints:

```bash
sudo docker run -it --rm \
  --memory="7g" \
  --memory-swap="23g" \
  --runtime=nvidia --network host \
  -v $HOME/.cache/huggingface:/root/.cache/huggingface \
  ghcr.io/nvidia-ai-iot/llama_cpp:gemma4-jetson-orin \
  llama-server -hf ggml-org/gemma-4-E4B-it-GGUF:Q4_K_M \
  -c 2048
```

The key flags:

- `--memory="7g"` — caps Docker's RAM usage at 7GB, leaving a small buffer for the system
- `--memory-swap="23g"` — total memory ceiling (7GB RAM + 16GB swap)
- `-c 2048` — limits the context window to 2048 tokens, a safe trade-off for 8GB hardware



## Performance

I was pleasantly surprised. Through the llama-server web interface, I'm getting **11–14 tokens per second** for standard generation. That's perfectly usable for reading along as text streams in — roughly the pace of comfortable reading. For general chatbot interaction, it feels responsive enough.

The reasoning step is where things slow down noticeably. Gemma 4's thinking mode adds a significant pause before the model starts generating the actual response. For a casual question-and-answer session, this is tolerable. For anything that requires chained tool calls or multi-step reasoning loops, the latency compounds quickly.

I had originally planned to use this Jetson as the backend for my [Meal Planning AI Agent](/projects/meal_planner_agent.html), which chains multiple MCP tool calls in a ReAct loop. In practice, the accumulated latency from repeated reasoning steps makes the agent too slow to be practical for that use case — at least with the current setup.

<!-- image: screenshot of llama-server web interface with generation stats -->


## What It's Actually Good For

The sweet spot for this setup is workloads that don't need real-time interaction. Applications where the model can run in the background, process something, and have a result ready whenever you check back. A few ideas I'm exploring:

- **News aggregation and summarization** — pulling RSS feeds overnight and generating daily briefings
- **Home automation reasoning** — connecting to [Home Assistant](https://www.home-assistant.io/) to handle more complex automation logic that goes beyond simple if/then rules
- **Autonomous coding agents** — I'm considering running an [OpenClaw](https://github.com/paracosm-ai/openclaw) agent, which can work asynchronously on coding tasks without needing a human in the loop

None of these require snappy back-and-forth. They're the kind of tasks where you kick something off, walk away, and come back later. That's where a low-power, always-on setup like this makes the most sense.


## Final Thoughts

For around the same power draw as a Raspberry Pi 5, I now have a dedicated LLM server that can run a surprisingly capable model 24/7. It's not going to replace a desktop GPU for heavy inference or interactive agent workflows, but it doesn't need to. It fills a different niche: a quiet, always-available box that can think in the background.

If you have a Jetson Orin Nano sitting around and haven't tried running an LLM on it yet, Gemma 4 E4B is a great model to start with. The NVIDIA Docker image makes the initial setup almost trivial, and with a bit of memory tuning, it runs stably within the 8GB constraint.

I'll update this post as I get more experience running real workloads on it. If you've tried a similar setup — or have ideas for good always-on LLM use cases — I'd be happy to hear about it.


## Resources

- [Gemma 4](https://deepmind.google/models/gemma/gemma-4/)
- [NVIDIA Jetson Orin Nano](https://www.nvidia.com/en-us/autonomous-machines/embedded-systems/jetson-orin/)
- [JetPack SDK / SDK Manager Installation Guide](https://docs.nvidia.com/sdk-manager/install-with-sdkm-jetson/index.html)
- [Gemma 4 on Jetson — NVIDIA Forum Thread](https://forums.developer.nvidia.com/t/no-luck-with-gemma-4-on-jetson-nano-super/365620/)

