# Benchmark Latency and Throughput

## SGLang

### Launch a server
```
python -m sglang.launch_server --model-path meta-llama/Llama-2-7b-chat-hf --port 30000
```

### Benchmark one batch

```
python3 bench_one.py
python3 bench_one.py --batch-size 64
```

### Benchmark online serving with many requests

```
python3 bench_serving.py --backend srt --port 30000 --tokenizer meta-llama/Llama-2-7b-chat-hf --num-prompt 1000 --request-rate 100 --input-len 1024 --output-len 256
```

### Benchmark online serving on the ShareGPT dataset

#### Download data
```
wget https://huggingface.co/datasets/anon8231489123/ShareGPT_Vicuna_unfiltered/resolve/main/ShareGPT_V3_unfiltered_cleaned_split.json
```

#### Run ShareGPT
```
python3 bench_serving.py --backend srt --port 30000 --tokenizer meta-llama/Llama-2-7b-chat-hf --dataset ShareGPT_V3_unfiltered_cleaned_split.json --num-prompts 10 --request-rate 10
```

### Profile with Nsight
0. Prerequisite
```bash
# install nsys
# https://docs.nvidia.com/nsight-systems/InstallationGuide/index.html
apt update
apt install -y --no-install-recommends gnupg
echo "deb http://developer.download.nvidia.com/devtools/repos/ubuntu$(source /etc/lsb-release; echo "$DISTRIB_RELEASE" | tr -d .)/$(dpkg --print-architecture) /" | tee /etc/apt/sources.list.d/nvidia-devtools.list
apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/7fa2af80.pub
apt update
apt install nsight-systems-cli
```

1. To profile a single batch, use `nsys profile --cuda-graph-trace=node python3 -m sglang.bench_latency --model meta-llama/Meta-Llama-3-8B --batch-size 64 --input-len 512`

2. To profile a server, e.g.

```bash
# server
# set the delay and duration times according to needs
nsys profile --trace-fork-before-exec=true --cuda-graph-trace=node -o sglang.out --delay 60 --duration 70 python3 -m sglang.launch_server --model-path meta-llama/Meta-Llama-3.1-8B-Instruct --disable-radix-cache

# client
python3 -m sglang.bench_serving --backend sglang --num-prompts 6000 --dataset-name random --random-input 4096 --random-output 2048
```

3. Use NVTX, e.g.

```bash
# install nvtx
pip install nvtx

# code snippets
import nvtx
with nvtx.annotate("description", color="color"):
    # some critical code
```


## Other baselines

### vLLM
```
python3 -m vllm.entrypoints.api_server --model meta-llama/Llama-2-7b-chat-hf --tensor-parallel 1 --disable-log-requests --swap-space 16 --port 21000
```

```
# run synthetic
python3 bench_serving.py --backend vllm --port 30000 --tokenizer meta-llama/Llama-2-7b-chat-hf --num-prompt 1000 --request-rate 100 --input-len 1024 --output-len 256
```

```
# run ShareGPT
python3 bench_serving.py --backend vllm --port 21000 --tokenizer meta-llama/Llama-2-7b-chat-hf --dataset ShareGPT_V3_unfiltered_cleaned_split.json --num-prompts 10 --request-rate 10
```

```
# run one batch
python3 -m vllm.entrypoints.openai.api_server --model meta-llama/Meta-Llama-3-70B --tensor 8 --disable-log-requests --max-num-seqs 1024 --quantization fp8

python3 bench_one.py --input-len 1024 --batch-size 1 1 2 4 8 16 32 64 128 256 512 768 1024 --port 8000 --backend vllm
```

### LightLLM
```
python -m lightllm.server.api_server --model_dir ~/model_weights/Llama-2-7b-chat-hf --max_total_token_num 15600 --tokenizer_mode auto --port 22000
```

```
python3 bench_serving.py --backend lightllm --port 22000 --tokenizer meta-llama/Llama-2-7b-chat-hf --dataset ShareGPT_V3_unfiltered_cleaned_split.json --num-prompts 10 --request-rate 10
```
