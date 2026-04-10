<!--
  CHAPTER: 57
  TITLE: Scaling & Cost at the Infra Level
  PART: 11 — AI Deployment & Infrastructure
  PHASE: 2 — Become an Expert
  PREREQS: Ch 47 (quantization), Ch 49 (cost engineering), Ch 53 (deployment), Ch 54 (gateway)
  KEY_TOPICS: GPU vs CPU inference, serverless inference, request batching, model caching, auto-scaling, CDN caching, self-host vs API break-even, quantized model serving, edge deployment, warm pools, cost modeling
  DIFFICULTY: Advanced
  LANGUAGE: TypeScript, Python, Terraform, Docker, YAML
  UPDATED: 2026-04-10
-->

# Chapter 57: Scaling & Cost at the Infra Level

> **Part 11 — AI Deployment & Infrastructure** | Phase 2: Become an Expert | Prerequisites: Ch 47, Ch 49, Ch 53, Ch 54 | Difficulty: Advanced

In Chapter 49, you learned to control costs at the application level: model routing, token optimization, prompt caching, usage budgets. In Chapter 53, you deployed to serverless functions and containers. In Chapter 54, you built a gateway with rate limiting and cost controls. Now it is time to think about cost and scale at the infrastructure level -- the decisions that determine whether your AI workload costs $500/month or $50,000/month.

Most teams never need to think about infrastructure-level scaling for AI. If you are calling OpenAI or Anthropic APIs, the provider handles the scaling. You pay per token, and the price is fixed. Your infrastructure cost (the serverless function or container that makes the API call) is a rounding error compared to the API cost.

But there are specific situations where infrastructure-level decisions matter enormously:

- You are serving your own models (open-source, fine-tuned, or quantized)
- You are processing millions of requests and the per-token cost at API providers is unsustainable
- You need sub-100ms latency that API providers cannot guarantee
- You are running batch inference jobs (embed 10 million documents, classify a data warehouse)
- You need to run models in a specific region or air-gapped environment for compliance

This chapter covers the infrastructure patterns for these situations: GPU vs CPU, serverless inference providers, request batching, model caching, auto-scaling, and the break-even math that tells you when to self-host.

### In This Chapter
- GPU vs CPU inference: when you actually need a GPU
- Serverless inference providers: Replicate, Modal, Together AI, SageMaker
- Batching requests for throughput
- Model caching and warm pools
- Auto-scaling patterns: scale on queue depth, not CPU
- CDN caching for static AI outputs
- The real cost model for AI infrastructure
- Self-host vs API: the break-even math with real numbers
- Quantized model serving: 4-bit models at a fraction of the cost
- Edge deployment: running small models closer to users

### Related Chapters
- **Ch 22 (Telemetry & Tracing)** -- spirals back: infrastructure metrics feed the telemetry pipeline
- **Ch 47 (Quantization & Deployment)** -- quantized models you serve here
- **Ch 48 (AI Security & Guardrails)** -- spirals back: scaling must not bypass security controls
- **Ch 49 (Cost Engineering)** -- application-level cost; now infrastructure-level
- **Ch 53 (Deploying LLM Applications)** -- the deployment targets you scale here
- **Ch 54 (API Gateway)** -- gateway-level controls that enable scaling patterns
- **Ch 55 (Sandboxing Agents)** -- sandbox infrastructure costs are part of the picture
- **Ch 59 (Building Internal AI Tools)** -- platform-scale infrastructure

---

## 1. GPU vs CPU Inference

### 1.1 When You Actually Need a GPU

The most expensive mistake in AI infrastructure is running GPU instances when you don't need them. Here is the decision framework:

```
Do you call an API provider (OpenAI, Anthropic, Google)?
  ├─ Yes → You do NOT need GPUs. The provider has them.
  │        Your infrastructure is just API calls.
  │        Skip to Section 2 (serverless inference) or Section 8 (break-even math).
  │
  └─ No → You are serving your own model.
          │
          ├─ Is it an embedding model (< 1B parameters)?
          │   ├─ Yes → CPU is often fine. Benchmark first.
          │   └─ No → Continue below.
          │
          ├─ Is it a small language model (< 3B parameters)?
          │   ├─ Yes → CPU may work. Quantized (4-bit) models run
          │   │        well on CPU for low-throughput use cases.
          │   └─ No → You probably need a GPU.
          │
          └─ Is it a large model (7B+ parameters)?
              └─ Yes → You need a GPU. Period.
                       How much GPU depends on:
                       - Model size (7B, 13B, 70B)
                       - Quantization level (FP16, INT8, INT4)
                       - Throughput requirement (requests/second)
                       - Latency requirement (time to first token)
```

### 1.2 GPU Instance Types and What They Cost

```
Instance Type        GPU          VRAM    Cost/hour   Good For
──────────────       ────         ────    ─────────   ────────
AWS g5.xlarge        A10G         24 GB   $1.01       7B models (FP16), 13B (INT4)
AWS g5.2xlarge       A10G         24 GB   $1.21       Same + more CPU/RAM
AWS g5.12xlarge      4x A10G      96 GB   $5.67       70B models (INT4)
AWS p4d.24xlarge     8x A100      320 GB  $32.77      Training, 70B+ (FP16)
AWS g6.xlarge        L4           24 GB   $0.80       7B models, best $/perf
AWS g6e.xlarge       L40S         48 GB   $1.86       13B (FP16), 70B (INT4)

GCP g2-standard-4    L4           24 GB   $0.74       7B models, embeddings
GCP a2-highgpu-1g    A100         40 GB   $3.67       13B+ models

Lambda Labs          A10          24 GB   $0.55       Budget option for dev
Together AI          Various      N/A     per-token   Serverless (no management)
Modal                Various      N/A     per-second  Serverless (pay for use)
Replicate            Various      N/A     per-second  Serverless (easiest)
```

### 1.3 VRAM Requirements by Model Size

The most critical constraint is VRAM. If the model doesn't fit in VRAM, it won't run (or it will run catastrophically slowly on CPU).

```
Model Size    FP16 (full)    INT8        INT4 (GPTQ/AWQ)
──────────    ───────────    ────        ────────────────
1B            2 GB           1 GB        0.5 GB
3B            6 GB           3 GB        1.5 GB
7B            14 GB          7 GB        3.5 GB
13B           26 GB          13 GB       6.5 GB
34B           68 GB          34 GB       17 GB
70B           140 GB         70 GB       35 GB

Rule of thumb: VRAM needed ≈ (parameters × bytes per parameter) + 20% overhead
  FP16: 2 bytes/param → 7B × 2 = 14 GB + overhead ≈ 17 GB
  INT4: 0.5 bytes/param → 7B × 0.5 = 3.5 GB + overhead ≈ 4.5 GB
```

### 1.4 CPU Inference: When It Works

CPU inference works well for:

```python
# CPU inference with llama-cpp-python
# Good for: embedding models, small models, low-throughput use cases
from llama_cpp import Llama

# 7B model quantized to 4-bit runs on any machine with 8GB RAM
llm = Llama(
    model_path="./models/llama-3.2-3b-instruct.Q4_K_M.gguf",
    n_ctx=4096,        # Context window
    n_threads=4,       # CPU threads to use
    n_gpu_layers=0,    # 0 = pure CPU inference
    verbose=False,
)

# On a modern CPU (e.g., AWS c7g.xlarge, 4 vCPU):
# - 3B Q4: ~15-20 tokens/sec (good enough for many use cases)
# - 7B Q4: ~8-12 tokens/sec (acceptable for non-interactive)
# - 13B Q4: ~3-5 tokens/sec (too slow for most uses)

output = llm.create_chat_completion(
    messages=[{"role": "user", "content": "Summarize this text..."}],
    max_tokens=256,
)
print(output["choices"][0]["message"]["content"])
```

**CPU vs GPU cost comparison for a 7B model:**

| Metric | CPU (c7g.2xlarge) | GPU (g6.xlarge) |
|--------|-------------------|-----------------|
| Cost/hour | $0.29 | $0.80 |
| Tokens/sec | ~15 (Q4) | ~100 (Q4) |
| Cost per 1M tokens | ~$5.40 | ~$2.20 |
| Best for | Low throughput, batch | High throughput, interactive |

For low-throughput use cases (< 100 requests/hour), CPU inference is 40% cheaper. For high-throughput use cases, GPU is 60% cheaper per token.

---

## 2. Serverless Inference Providers

### 2.1 When Serverless Inference Makes Sense

Serverless inference providers run models for you. You don't manage instances, GPUs, or model loading. You pay per request or per second of GPU time.

**Use serverless inference when:**
- Your traffic is bursty (quiet periods → spikes)
- You don't want to manage GPU infrastructure
- You need to test multiple models without committing to infrastructure
- Your volume doesn't justify dedicated GPU instances (< 1M tokens/day)

### 2.2 Provider Comparison

```
Provider       Models              Pricing Model     Cold Start    Best For
────────       ──────              ─────────────     ──────────    ────────
Replicate      Open-source         Per-second GPU    5-30s         Prototyping, image gen
Modal          Any (bring your     Per-second GPU    1-5s          Custom models, low latency
               own or theirs)
Together AI    Open-source LLMs    Per-token         0s (warm)     Text generation, cheap
SageMaker      Any (deploy your    Per-hour          30-60s        Enterprise, AWS ecosystem
  Endpoints    own)
Fireworks AI   Open-source LLMs    Per-token         0s (warm)     Low-latency text gen
```

### 2.3 Replicate: Simplest Path

```python
# Replicate: run any open-source model with one API call
import replicate

# Run Llama 3 (serverless — you don't manage any infrastructure)
output = replicate.run(
    "meta/meta-llama-3.1-70b-instruct",
    input={
        "prompt": "Explain quantum computing in simple terms.",
        "max_tokens": 512,
        "temperature": 0.7,
    },
)
# output is a generator — iterate for streaming
result = "".join(output)
print(result)

# Run an image model
output = replicate.run(
    "stability-ai/sdxl:latest",
    input={
        "prompt": "A futuristic city at sunset, digital art",
        "width": 1024,
        "height": 1024,
    },
)
# output is a list of URLs to the generated images
print(output[0])  # URL to the image
```

**Cost example:** Replicate charges per-second of GPU time.
- Llama 3.1 70B on A40: ~$0.0023/sec
- Average request (5 seconds): ~$0.012
- 10,000 requests/day: ~$120/day = ~$3,600/month

### 2.4 Modal: Low-Latency Custom Models

```python
# modal_app.py — deploy a custom model on Modal
import modal

app = modal.App("llm-inference")

# Define the GPU-equipped container
image = (
    modal.Image.debian_slim(python_version="3.11")
    .pip_install("vllm==0.6.0", "torch==2.4.0")
)

@app.cls(
    image=image,
    gpu=modal.gpu.A10G(),          # 24 GB VRAM
    container_idle_timeout=300,     # Keep warm for 5 min after last request
    allow_concurrent_inputs=10,     # Handle 10 concurrent requests
)
class LLMInference:
    @modal.enter()
    def load_model(self):
        """Load model once when container starts."""
        from vllm import LLM
        self.llm = LLM(
            model="meta-llama/Llama-3.1-8B-Instruct",
            quantization="awq",    # 4-bit quantization
            max_model_len=4096,
            gpu_memory_utilization=0.9,
        )

    @modal.method()
    def generate(self, prompt: str, max_tokens: int = 512) -> str:
        from vllm import SamplingParams
        params = SamplingParams(
            max_tokens=max_tokens,
            temperature=0.7,
        )
        outputs = self.llm.generate([prompt], params)
        return outputs[0].outputs[0].text

    @modal.method()
    def batch_generate(self, prompts: list[str], max_tokens: int = 512) -> list[str]:
        """Process multiple prompts in a single batch for throughput."""
        from vllm import SamplingParams
        params = SamplingParams(max_tokens=max_tokens, temperature=0.7)
        outputs = self.llm.generate(prompts, params)
        return [o.outputs[0].text for o in outputs]


# Deploy and use
@app.local_entrypoint()
def main():
    model = LLMInference()
    result = model.generate.remote("Explain quantum computing simply.")
    print(result)
    
    # Batch: process 50 prompts at once
    prompts = [f"Summarize this document: {doc}" for doc in documents]
    results = model.batch_generate.remote(prompts)
```

**Cost example:** Modal charges per-second of GPU time.
- A10G: ~$0.000575/sec ($2.07/hour)
- Container idle: same rate (set `container_idle_timeout` wisely)
- 10,000 requests/day, ~2 sec each: ~$11.50/day = ~$345/month
- vs. Dedicated A10G (24/7): ~$1,490/month

Modal is 4x cheaper when utilization is under ~25%.

### 2.5 Together AI: Token-Based Pricing

```typescript
// Together AI: OpenAI-compatible API for open-source models
import { generateText } from "ai";
import { createOpenAI } from "@ai-sdk/openai";

// Together AI uses the OpenAI-compatible API format
const together = createOpenAI({
  apiKey: process.env.TOGETHER_API_KEY!,
  baseURL: "https://api.together.xyz/v1",
});

const result = await generateText({
  model: together("meta-llama/Llama-3.1-70B-Instruct-Turbo"),
  prompt: "Explain quantum computing in simple terms.",
  maxTokens: 512,
});

console.log(result.text);
// Cost: ~$0.88/M tokens (Llama 3.1 70B Turbo)
// vs Claude Sonnet at ~$9/M tokens blended
// That is 10x cheaper for workloads where Llama 70B is sufficient
```

---

## 3. Batching Requests

### 3.1 Why Batching Matters

GPU inference has high fixed cost (loading the model) and low marginal cost (processing additional tokens in parallel). Processing 10 prompts in one batch is nearly as fast as processing 1 prompt, but 10x the throughput.

### 3.2 Request Batching Service

```python
# batch_service.py — batch incoming requests for GPU efficiency
import asyncio
import time
from dataclasses import dataclass, field
from typing import Optional
from collections import deque


@dataclass
class BatchRequest:
    id: str
    prompt: str
    max_tokens: int = 512
    result: Optional[str] = None
    error: Optional[str] = None
    event: asyncio.Event = field(default_factory=asyncio.Event)
    created_at: float = field(default_factory=time.time)


class BatchProcessor:
    """Accumulate requests and process them in GPU-efficient batches."""

    def __init__(
        self,
        model_fn,               # Function that takes a list of prompts
        max_batch_size: int = 32,
        max_wait_ms: int = 100,  # Max time to wait for batch to fill
    ):
        self.model_fn = model_fn
        self.max_batch_size = max_batch_size
        self.max_wait_ms = max_wait_ms
        self.queue: deque[BatchRequest] = deque()
        self._lock = asyncio.Lock()
        self._processing = False

    async def submit(self, prompt: str, max_tokens: int = 512) -> str:
        """Submit a request and wait for the result."""
        request = BatchRequest(
            id=f"req-{time.time_ns()}",
            prompt=prompt,
            max_tokens=max_tokens,
        )

        async with self._lock:
            self.queue.append(request)

        # Trigger batch processing
        asyncio.create_task(self._maybe_process())

        # Wait for result
        await request.event.wait()

        if request.error:
            raise RuntimeError(request.error)
        return request.result

    async def _maybe_process(self):
        """Process a batch if we have enough requests or waited long enough."""
        if self._processing:
            return

        # Wait a short time for more requests to accumulate
        await asyncio.sleep(self.max_wait_ms / 1000)

        async with self._lock:
            if self._processing or len(self.queue) == 0:
                return
            self._processing = True

            # Take up to max_batch_size requests
            batch: list[BatchRequest] = []
            while self.queue and len(batch) < self.max_batch_size:
                batch.append(self.queue.popleft())

        try:
            # Process the entire batch at once on the GPU
            prompts = [r.prompt for r in batch]
            max_tokens = max(r.max_tokens for r in batch)

            start = time.time()
            results = await asyncio.to_thread(
                self.model_fn, prompts, max_tokens
            )
            elapsed = time.time() - start

            print(
                f"Batch processed: {len(batch)} requests in {elapsed:.2f}s "
                f"({len(batch)/elapsed:.1f} req/s)"
            )

            # Distribute results
            for request, result in zip(batch, results):
                request.result = result
                request.event.set()

        except Exception as e:
            for request in batch:
                request.error = str(e)
                request.event.set()

        finally:
            self._processing = False
            # Process next batch if queue is not empty
            if self.queue:
                asyncio.create_task(self._maybe_process())


# Usage with vLLM
def create_vllm_batch_fn():
    from vllm import LLM, SamplingParams

    llm = LLM(
        model="meta-llama/Llama-3.1-8B-Instruct",
        quantization="awq",
        max_model_len=4096,
    )

    def batch_generate(prompts: list[str], max_tokens: int) -> list[str]:
        params = SamplingParams(max_tokens=max_tokens, temperature=0.7)
        outputs = llm.generate(prompts, params)
        return [o.outputs[0].text for o in outputs]

    return batch_generate


# FastAPI integration
from fastapi import FastAPI

app = FastAPI()
batch_processor = BatchProcessor(
    model_fn=create_vllm_batch_fn(),
    max_batch_size=32,
    max_wait_ms=100,  # Wait up to 100ms for batch to fill
)


@app.post("/generate")
async def generate(request: dict):
    result = await batch_processor.submit(
        prompt=request["prompt"],
        max_tokens=request.get("max_tokens", 512),
    )
    return {"text": result}
```

### 3.3 Batching Impact

```
Without batching (sequential):
  32 requests × 2 seconds each = 64 seconds total
  Throughput: 0.5 requests/second

With batching (batch size 32):
  32 requests processed in ~3 seconds
  Throughput: ~10 requests/second
  
  That is a 20x improvement in throughput.
  GPU utilization goes from ~5% to ~90%.
```

---

## 4. Model Caching and Warm Pools

### 4.1 The Cold Start Problem for Model Serving

Loading a model into GPU memory takes time:

```
Model Size     Load Time (SSD → GPU)    Load Time (S3 → GPU)
──────────     ─────────────────────    ────────────────────
1B (Q4)        2-5 seconds              15-30 seconds
7B (Q4)        5-10 seconds             30-60 seconds
13B (Q4)       10-20 seconds            60-120 seconds
70B (Q4)       30-60 seconds            3-5 minutes
```

For interactive workloads, these cold starts are unacceptable. You need warm pools.

### 4.2 Warm Pool Architecture

```python
# warm_pool.py — keep models loaded and ready
import asyncio
from dataclasses import dataclass
from typing import Optional
from datetime import datetime, timedelta


@dataclass
class WarmInstance:
    model_id: str
    instance_id: str
    loaded_at: datetime
    last_used: datetime
    requests_served: int
    healthy: bool


class WarmPool:
    """Maintain a pool of GPU instances with models pre-loaded."""

    def __init__(
        self,
        min_instances: int = 1,
        max_instances: int = 10,
        idle_timeout_minutes: int = 30,
        scale_up_threshold: float = 0.7,  # Scale up when 70% of pool is busy
    ):
        self.min_instances = min_instances
        self.max_instances = max_instances
        self.idle_timeout = timedelta(minutes=idle_timeout_minutes)
        self.scale_up_threshold = scale_up_threshold
        self.instances: list[WarmInstance] = []
        self.busy_instances: set[str] = set()

    def get_instance(self) -> Optional[WarmInstance]:
        """Get an available warm instance."""
        available = [
            i for i in self.instances
            if i.instance_id not in self.busy_instances and i.healthy
        ]

        if not available:
            return None

        # Pick the most recently used (better cache locality)
        instance = max(available, key=lambda i: i.last_used)
        self.busy_instances.add(instance.instance_id)
        return instance

    def release_instance(self, instance_id: str):
        """Release an instance back to the pool."""
        self.busy_instances.discard(instance_id)
        for instance in self.instances:
            if instance.instance_id == instance_id:
                instance.last_used = datetime.now()
                instance.requests_served += 1

    def should_scale_up(self) -> bool:
        """Check if we need more instances."""
        if len(self.instances) >= self.max_instances:
            return False
        busy_ratio = len(self.busy_instances) / max(len(self.instances), 1)
        return busy_ratio >= self.scale_up_threshold

    def should_scale_down(self) -> list[str]:
        """Find instances that should be terminated."""
        if len(self.instances) <= self.min_instances:
            return []

        idle = [
            i for i in self.instances
            if i.instance_id not in self.busy_instances
            and datetime.now() - i.last_used > self.idle_timeout
        ]

        # Keep at least min_instances
        removable = len(self.instances) - self.min_instances
        return [i.instance_id for i in idle[:removable]]

    def get_metrics(self) -> dict:
        return {
            "total_instances": len(self.instances),
            "busy_instances": len(self.busy_instances),
            "utilization": len(self.busy_instances) / max(len(self.instances), 1),
            "total_requests_served": sum(i.requests_served for i in self.instances),
        }
```

### 4.3 Terraform for GPU Warm Pool

```hcl
# terraform/gpu-pool.tf — auto-scaling GPU instances with pre-loaded models

# Launch template with model pre-loading
resource "aws_launch_template" "gpu_inference" {
  name_prefix   = "gpu-inference-"
  image_id      = data.aws_ami.deep_learning.id
  instance_type = "g6.xlarge"  # L4 GPU, 24 GB VRAM

  block_device_mappings {
    device_name = "/dev/sda1"
    ebs {
      volume_size           = 100  # GB — for model storage
      volume_type           = "gp3"
      iops                  = 4000
      throughput            = 250
      delete_on_termination = true
    }
  }

  user_data = base64encode(<<-EOF
    #!/bin/bash
    set -e
    
    # Download model from S3 on boot
    aws s3 cp s3://${var.model_bucket}/models/llama-3.1-8b-instruct-awq/ \
      /opt/models/llama-3.1-8b/ --recursive
    
    # Start the inference server
    docker run -d \
      --gpus all \
      --name inference \
      -p 8000:8000 \
      -v /opt/models:/models \
      -e MODEL_PATH=/models/llama-3.1-8b \
      -e MAX_MODEL_LEN=4096 \
      -e GPU_MEMORY_UTILIZATION=0.9 \
      ${aws_ecr_repository.inference.repository_url}:latest
    
    # Signal that the instance is ready
    /opt/aws/bin/cfn-signal -e $? --stack ${var.stack_name} \
      --resource GPUAutoScalingGroup --region ${var.aws_region}
    EOF
  )

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "gpu-inference"
      Role = "model-serving"
    }
  }
}

# Auto-scaling group
resource "aws_autoscaling_group" "gpu_inference" {
  name                = "gpu-inference-asg"
  desired_capacity    = var.gpu_min_instances
  min_size            = var.gpu_min_instances
  max_size            = var.gpu_max_instances
  vpc_zone_identifier = var.private_subnet_ids

  launch_template {
    id      = aws_launch_template.gpu_inference.id
    version = "$Latest"
  }

  # Health check: verify the model is loaded and responding
  health_check_type         = "ELB"
  health_check_grace_period = 300  # 5 min to load model

  target_group_arns = [aws_lb_target_group.inference.arn]

  tag {
    key                 = "Name"
    value               = "gpu-inference"
    propagate_at_launch = true
  }
}

# Scale based on queue depth (NOT CPU utilization)
resource "aws_autoscaling_policy" "scale_on_queue" {
  name                   = "scale-on-queue-depth"
  autoscaling_group_name = aws_autoscaling_group.gpu_inference.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    customized_metric_specification {
      metric_name = "ApproximateNumberOfMessagesVisible"
      namespace   = "AWS/SQS"
      statistic   = "Average"
      dimensions {
        name  = "QueueName"
        value = aws_sqs_queue.inference_requests.name
      }
    }
    target_value = 10  # Scale up when queue depth > 10 per instance
  }
}
```

---

## 5. Auto-Scaling Patterns

### 5.1 Scale on Queue Depth, Not CPU

For AI workloads, CPU utilization is a terrible scaling metric. A GPU instance running inference is 90% idle on the CPU while the GPU is 100% utilized. CPU-based auto-scaling will never trigger.

```
WRONG metric: CPU utilization
  GPU instance CPU: 5% (doing nothing)
  GPU utilization: 95% (maxed out)
  Auto-scaler thinks: "Low CPU, no need to scale"
  Reality: requests are queuing, latency is spiking

RIGHT metric: Queue depth / Request queue time
  Queue depth: 50 messages
  Auto-scaler thinks: "Queue is growing, add instances"
  Reality: new instance spins up, loads model, starts processing
```

### 5.2 Scaling Metrics for AI Workloads

```hcl
# terraform/scaling.tf — scaling metrics for AI inference

# CloudWatch alarms for auto-scaling decisions
resource "aws_cloudwatch_metric_alarm" "queue_depth_high" {
  alarm_name          = "inference-queue-depth-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  period              = 60
  statistic           = "Average"
  threshold           = 50
  alarm_description   = "Inference queue depth is high"
  alarm_actions       = [aws_autoscaling_policy.scale_up.arn]

  dimensions = {
    QueueName = aws_sqs_queue.inference_requests.name
  }
}

resource "aws_cloudwatch_metric_alarm" "p99_latency_high" {
  alarm_name          = "inference-p99-latency-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "P99Latency"
  namespace           = "AIInference"
  period              = 60
  statistic           = "Average"
  threshold           = 10000  # 10 seconds
  alarm_description   = "P99 inference latency is high"
  alarm_actions       = [aws_autoscaling_policy.scale_up.arn]
}

# Scale-down is slower than scale-up (model loading takes time)
resource "aws_autoscaling_policy" "scale_up" {
  name                   = "inference-scale-up"
  autoscaling_group_name = aws_autoscaling_group.gpu_inference.name
  adjustment_type        = "ChangeInCapacity"
  scaling_adjustment     = 2       # Add 2 instances at a time
  cooldown               = 300     # 5 min cooldown (model load time)
}

resource "aws_autoscaling_policy" "scale_down" {
  name                   = "inference-scale-down"
  autoscaling_group_name = aws_autoscaling_group.gpu_inference.name
  adjustment_type        = "ChangeInCapacity"
  scaling_adjustment     = -1      # Remove 1 at a time (conservative)
  cooldown               = 600     # 10 min cooldown
}
```

### 5.3 Predictive Scaling

If your traffic is predictable (peaks during business hours, quiet at night), use scheduled scaling:

```hcl
# terraform/scheduled-scaling.tf — predictive scaling for known traffic patterns

# Scale up before business hours (models take 5+ min to load)
resource "aws_autoscaling_schedule" "morning_scaleup" {
  scheduled_action_name  = "morning-scaleup"
  autoscaling_group_name = aws_autoscaling_group.gpu_inference.name
  min_size               = 3
  max_size               = 10
  desired_capacity       = 5
  recurrence             = "0 7 * * MON-FRI"  # 7 AM weekdays
  time_zone              = "America/New_York"
}

# Scale down after business hours
resource "aws_autoscaling_schedule" "evening_scaledown" {
  scheduled_action_name  = "evening-scaledown"
  autoscaling_group_name = aws_autoscaling_group.gpu_inference.name
  min_size               = 1
  max_size               = 5
  desired_capacity       = 1
  recurrence             = "0 20 * * MON-FRI"  # 8 PM weekdays
  time_zone              = "America/New_York"
}

# Weekend: minimal capacity
resource "aws_autoscaling_schedule" "weekend" {
  scheduled_action_name  = "weekend-minimal"
  autoscaling_group_name = aws_autoscaling_group.gpu_inference.name
  min_size               = 1
  max_size               = 3
  desired_capacity       = 1
  recurrence             = "0 20 * * FRI"  # Friday 8 PM
  time_zone              = "America/New_York"
}
```

---

## 6. CDN Caching for Static AI Outputs

### 6.1 What Can Be Cached

Some AI outputs are static -- they don't change for a given input. These can be served from a CDN.

| Output Type | Cacheable? | TTL | Example |
|------------|-----------|-----|---------|
| Generated images | Yes | Days to forever | Product image variations |
| Pre-computed embeddings | Yes | Until document changes | Document search indexes |
| Summarizations of static docs | Yes | Until document changes | Help article summaries |
| Transcriptions | Yes | Forever | Audio/video transcripts |
| Classifications | Mostly | Hours to days | Content categorization |
| Chat responses | No | N/A | Personalized, dynamic |

### 6.2 CDN Configuration for AI Outputs

```hcl
# terraform/cdn.tf — CloudFront for AI-generated static assets

resource "aws_cloudfront_distribution" "ai_assets" {
  origin {
    domain_name = aws_s3_bucket.ai_outputs.bucket_regional_domain_name
    origin_id   = "ai-outputs-s3"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.ai.cloudfront_access_identity_path
    }
  }

  enabled         = true
  is_ipv6_enabled = true
  comment         = "AI-generated static assets"

  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "ai-outputs-s3"
    viewer_protocol_policy = "redirect-to-https"

    cache_policy_id = aws_cloudfront_cache_policy.ai_assets.id

    compress = true
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}

resource "aws_cloudfront_cache_policy" "ai_assets" {
  name        = "ai-assets-cache"
  comment     = "Cache policy for AI-generated assets"
  default_ttl = 86400    # 1 day
  max_ttl     = 2592000  # 30 days
  min_ttl     = 3600     # 1 hour

  parameters_in_cache_key_and_forwarded_to_origin {
    cookies_config {
      cookie_behavior = "none"
    }
    headers_config {
      header_behavior = "none"
    }
    query_strings_config {
      query_string_behavior = "none"
    }
  }
}
```

### 6.3 Pre-Computing AI Outputs

```python
# scripts/precompute_embeddings.py — batch generate and cache embeddings
import asyncio
import json
import boto3
from sentence_transformers import SentenceTransformer


async def precompute_embeddings(
    documents: list[dict],
    model_name: str = "all-MiniLM-L6-v2",
    batch_size: int = 256,
    s3_bucket: str = "ai-outputs",
    s3_prefix: str = "embeddings/",
):
    """Pre-compute embeddings for all documents and store in S3."""
    model = SentenceTransformer(model_name)
    s3 = boto3.client("s3")

    total = len(documents)
    print(f"Computing embeddings for {total} documents...")

    for i in range(0, total, batch_size):
        batch = documents[i : i + batch_size]
        texts = [doc["content"] for doc in batch]

        # Batch encode (GPU-accelerated if available)
        embeddings = model.encode(
            texts,
            batch_size=batch_size,
            show_progress_bar=False,
            normalize_embeddings=True,
        )

        # Store each embedding in S3
        for doc, embedding in zip(batch, embeddings):
            key = f"{s3_prefix}{doc['id']}.json"
            s3.put_object(
                Bucket=s3_bucket,
                Key=key,
                Body=json.dumps({
                    "document_id": doc["id"],
                    "embedding": embedding.tolist(),
                    "model": model_name,
                    "dimensions": len(embedding),
                }),
                ContentType="application/json",
                # CDN-friendly cache headers
                CacheControl="public, max-age=2592000",  # 30 days
            )

        print(f"  Processed {min(i + batch_size, total)}/{total}")

    print(f"Done. {total} embeddings stored in s3://{s3_bucket}/{s3_prefix}")
```

---

## 7. The Real Cost Model

### 7.1 Total Cost of AI Infrastructure

The per-token or per-request cost is only part of the picture. The total cost includes:

```
Total Monthly Cost = Tokens + Compute + Storage + Bandwidth + Engineering

Where:
  Tokens     = API calls to LLM providers (often the biggest cost)
  Compute    = Servers, GPUs, serverless functions
  Storage    = Vector databases, S3, model weights, logs
  Bandwidth  = Data transfer, CDN, API responses
  Engineering = Your time managing the infrastructure
```

### 7.2 Cost Breakdown Example

A real-world AI application serving 100,000 daily active users:

```
Component                 Monthly Cost     % of Total
─────────                 ────────────     ──────────
LLM API (Claude Sonnet)   $8,500           62%
  - 5M requests/month
  - ~200 tokens avg input
  - ~500 tokens avg output
  - $3/M input, $15/M output

Vector Database (Pinecone) $700             5%
  - 2M vectors
  - s1 pod

Vercel (Pro)               $20              0.1%
  - Functions for API routes

Redis (Upstash)            $50              0.4%
  - Rate limiting, caching

Monitoring (Datadog)       $200             1.5%
  - APM + LLM monitoring

Eval Costs                 $300             2%
  - CI evals + scheduled evals

Engineering Time           $4,000           29%
  - ~10 hours/month maintenance
  - $400/hour fully loaded cost

TOTAL                      ~$13,770/month
```

Notice: **62% of the cost is the LLM API.** The infrastructure is a rounding error. This is why Chapter 49 (cost engineering at the application level) is far more impactful than infrastructure optimization for most teams.

### 7.3 Where Infrastructure Cost Dominates

Infrastructure cost becomes dominant when you self-host models:

```
Component                 Self-Hosted        API Provider
─────────                 ───────────        ────────────
LLM Inference             $3,500/month       $8,500/month
  (GPU instances vs API)  (3x g6.xlarge)     (Claude Sonnet API)

Infrastructure Management $2,000/month       $0
  (Engineer time)         (~5 hrs/month)

Model Updates             $500/month         $0
  (Testing, deploying)

GPU Idle Cost             $800/month         $0
  (Nights, weekends)

Total                     $6,800/month       $8,500/month

Savings                   $1,700/month (20%)
```

The savings are real but modest. And they come with significant operational burden. This is the break-even analysis from the next section.

---

## 8. Self-Host vs API: The Break-Even Math

### 8.1 The Decision Framework

```
Use API providers when:
  ✓ Volume < 5M tokens/day
  ✓ You don't have GPU operations expertise
  ✓ Latency requirements are > 500ms time-to-first-token
  ✓ You need the latest models immediately
  ✓ Your team is < 5 engineers

Self-host when:
  ✓ Volume > 20M tokens/day
  ✓ You have ML/infrastructure engineers
  ✓ You need consistent sub-100ms latency
  ✓ Compliance requires on-premises or specific region
  ✓ A fine-tuned open-source model meets your quality bar
  ✓ You're running batch workloads (embeddings, classification)
```

### 8.2 The Break-Even Calculator

```python
# scripts/break_even.py — calculate self-host vs API break-even point
from dataclasses import dataclass


@dataclass
class APIProviderCost:
    name: str
    input_price_per_1m: float   # $ per 1M input tokens
    output_price_per_1m: float  # $ per 1M output tokens


@dataclass
class SelfHostCost:
    gpu_instance_cost_per_hour: float
    num_instances: int
    throughput_tokens_per_second: float  # Total across all instances
    engineer_hours_per_month: float
    engineer_hourly_rate: float
    storage_monthly: float  # S3, EBS for models
    networking_monthly: float


def calculate_break_even(
    api: APIProviderCost,
    self_host: SelfHostCost,
    avg_input_tokens: int,
    avg_output_tokens: int,
) -> dict:
    """Find the monthly request volume where self-hosting breaks even."""

    # API cost per request
    api_cost_per_request = (
        (avg_input_tokens / 1_000_000) * api.input_price_per_1m +
        (avg_output_tokens / 1_000_000) * api.output_price_per_1m
    )

    # Self-host fixed monthly cost
    gpu_monthly = self_host.gpu_instance_cost_per_hour * 24 * 30 * self_host.num_instances
    engineer_monthly = self_host.engineer_hours_per_month * self_host.engineer_hourly_rate
    fixed_monthly = gpu_monthly + engineer_monthly + self_host.storage_monthly + self_host.networking_monthly

    # Self-host max throughput (requests per month)
    tokens_per_request = avg_input_tokens + avg_output_tokens
    max_requests_per_second = self_host.throughput_tokens_per_second / tokens_per_request
    max_requests_per_month = max_requests_per_second * 86400 * 30

    # Break-even: fixed_monthly / api_cost_per_request
    if api_cost_per_request <= 0:
        return {"error": "API cost per request is zero"}

    break_even_requests = fixed_monthly / api_cost_per_request

    return {
        "api_cost_per_request": api_cost_per_request,
        "api_cost_per_1k_requests": api_cost_per_request * 1000,
        "self_host_fixed_monthly": fixed_monthly,
        "self_host_gpu_monthly": gpu_monthly,
        "self_host_engineer_monthly": engineer_monthly,
        "break_even_requests_per_month": int(break_even_requests),
        "break_even_requests_per_day": int(break_even_requests / 30),
        "max_throughput_per_month": int(max_requests_per_month),
        "utilization_at_break_even": break_even_requests / max_requests_per_month,
    }


# Example calculations
if __name__ == "__main__":
    # Scenario 1: Claude Sonnet vs Llama 3.1 70B on GPU
    result = calculate_break_even(
        api=APIProviderCost("Claude Sonnet", input_price_per_1m=3.0, output_price_per_1m=15.0),
        self_host=SelfHostCost(
            gpu_instance_cost_per_hour=5.67,  # g5.12xlarge (4x A10G)
            num_instances=2,
            throughput_tokens_per_second=200,  # ~200 tok/s on 4x A10G
            engineer_hours_per_month=10,
            engineer_hourly_rate=150,
            storage_monthly=50,
            networking_monthly=30,
        ),
        avg_input_tokens=500,
        avg_output_tokens=300,
    )

    print("Scenario 1: Claude Sonnet vs Self-Hosted Llama 70B")
    print(f"  API cost per request: ${result['api_cost_per_request']:.4f}")
    print(f"  Self-host fixed monthly: ${result['self_host_fixed_monthly']:.0f}")
    print(f"  Break-even: {result['break_even_requests_per_day']:,} requests/day")
    print(f"  Break-even: {result['break_even_requests_per_month']:,} requests/month")
    print()

    # Scenario 2: GPT-4o-mini vs Llama 3.1 8B on single GPU
    result2 = calculate_break_even(
        api=APIProviderCost("GPT-4o-mini", input_price_per_1m=0.15, output_price_per_1m=0.6),
        self_host=SelfHostCost(
            gpu_instance_cost_per_hour=0.80,  # g6.xlarge (L4)
            num_instances=1,
            throughput_tokens_per_second=100,  # ~100 tok/s on L4
            engineer_hours_per_month=5,
            engineer_hourly_rate=150,
            storage_monthly=20,
            networking_monthly=10,
        ),
        avg_input_tokens=500,
        avg_output_tokens=300,
    )

    print("Scenario 2: GPT-4o-mini vs Self-Hosted Llama 8B")
    print(f"  API cost per request: ${result2['api_cost_per_request']:.6f}")
    print(f"  Self-host fixed monthly: ${result2['self_host_fixed_monthly']:.0f}")
    print(f"  Break-even: {result2['break_even_requests_per_day']:,} requests/day")
    print(f"  Break-even: {result2['break_even_requests_per_month']:,} requests/month")
```

**Expected output:**

```
Scenario 1: Claude Sonnet vs Self-Hosted Llama 70B
  API cost per request: $0.0060
  Self-host fixed monthly: $9,744
  Break-even: 54,133 requests/day
  Break-even: 1,624,000 requests/month

Scenario 2: GPT-4o-mini vs Self-Hosted Llama 8B
  API cost per request: $0.000255
  Self-host fixed monthly: $1,376
  Break-even: 179,739 requests/day
  Break-even: 5,392,157 requests/month
```

**Key insight:** Self-hosting only makes financial sense at very high volumes. For Scenario 1, you need 54,000+ requests/day before self-hosting is cheaper. For cheap API models like GPT-4o-mini, the break-even is nearly 180,000 requests/day.

### 8.3 The Hidden Costs of Self-Hosting

The break-even calculator doesn't capture everything:

```
Hidden Costs of Self-Hosting:
  - Model updates (Llama 3.1 → 3.2 → 4): testing, deployment, evals
  - On-call for GPU infrastructure (GPU failures, OOM errors)
  - Security patching for inference servers
  - Monitoring and alerting setup
  - Multi-region deployment for redundancy
  - No access to latest proprietary models (Claude, GPT-5)
  - Quality gap: Llama 70B ≠ Claude Sonnet for all tasks

Hidden Benefits of Self-Hosting:
  - Predictable costs (no surprise bills)
  - No rate limits from providers
  - Full control over latency and SLAs
  - Data never leaves your infrastructure
  - Can fine-tune and customize models
  - No vendor lock-in
```

---

## 9. Quantized Model Serving

### 9.1 Why Quantization Matters for Serving

From Chapter 47, quantization reduces model size by using lower-precision numbers. For serving, this means:

- **Less VRAM needed** -- 4-bit quantized 7B model fits in 4 GB (vs 14 GB at FP16)
- **Faster inference** -- less data to move through GPU memory bandwidth
- **Cheaper instances** -- can use smaller GPUs
- **Minimal quality loss** -- 4-bit quantization loses <2% quality on most benchmarks

### 9.2 Serving Quantized Models with vLLM

```python
# serve_quantized.py — serve a quantized model with vLLM
from vllm import LLM, SamplingParams
from vllm.entrypoints.openai.api_server import app
import uvicorn


def create_server(
    model: str = "TheBloke/Llama-2-7B-Chat-AWQ",
    quantization: str = "awq",
    gpu_memory_utilization: float = 0.9,
    max_model_len: int = 4096,
):
    """Create a vLLM inference server with quantized model."""
    
    llm = LLM(
        model=model,
        quantization=quantization,
        gpu_memory_utilization=gpu_memory_utilization,
        max_model_len=max_model_len,
        # Enable continuous batching for throughput
        enable_prefix_caching=True,
        # Tensor parallelism for multi-GPU
        tensor_parallel_size=1,  # Set to number of GPUs
    )
    
    return llm


# Dockerfile for quantized model serving
"""
FROM nvidia/cuda:12.1.0-runtime-ubuntu22.04

RUN apt-get update && apt-get install -y python3 python3-pip && \
    rm -rf /var/lib/apt/lists/*

RUN pip3 install vllm==0.6.0

# Download model at build time (cached in Docker layer)
RUN python3 -c "from huggingface_hub import snapshot_download; \
    snapshot_download('TheBloke/Llama-2-7B-Chat-AWQ', local_dir='/models/llama-7b-awq')"

EXPOSE 8000

CMD ["python3", "-m", "vllm.entrypoints.openai.api_server", \
     "--model", "/models/llama-7b-awq", \
     "--quantization", "awq", \
     "--max-model-len", "4096", \
     "--host", "0.0.0.0", \
     "--port", "8000"]
"""
```

### 9.3 Quantization Impact on Cost

```
Model: Llama 3.1 8B
Instance: AWS g6.xlarge (L4, 24 GB VRAM)
Cost: $0.80/hour

FP16 (full precision):
  VRAM needed: ~17 GB
  Throughput: ~60 tokens/sec
  Cost per 1M tokens: $3.70

INT8 (8-bit):
  VRAM needed: ~9 GB
  Throughput: ~85 tokens/sec
  Cost per 1M tokens: $2.60

INT4 (AWQ/GPTQ):
  VRAM needed: ~5 GB
  Throughput: ~110 tokens/sec
  Cost per 1M tokens: $2.02

Savings from FP16 → INT4: 45% cost reduction with < 2% quality loss
```

---

## 10. Edge Deployment

### 10.1 Running Small Models Closer to Users

For specific use cases (classification, simple extraction, embedding), you can run small models at the edge -- in CDN edge locations or on user devices.

**Good candidates for edge deployment:**
- Text classification (< 100M parameters)
- Named entity recognition
- Sentiment analysis
- Simple embeddings (e.g., all-MiniLM-L6-v2 at 23M parameters)
- Autocomplete / suggestions

**Not suitable for edge:**
- Large language models (> 1B parameters)
- Image generation
- Complex reasoning tasks
- Multi-turn conversations

### 10.2 Edge Inference with ONNX Runtime

```typescript
// lib/edge-inference.ts — run small models on the edge
// Works in Vercel Edge Functions, Cloudflare Workers, Deno Deploy

import * as ort from "onnxruntime-node";

let session: ort.InferenceSession | null = null;

async function getSession(): Promise<ort.InferenceSession> {
  if (!session) {
    // Load a small classification model (ONNX format)
    // The model file is deployed with the edge function
    session = await ort.InferenceSession.create("./models/classifier.onnx", {
      executionProviders: ["cpu"], // Edge has no GPU
    });
  }
  return session;
}

export async function classifyText(text: string): Promise<{
  label: string;
  confidence: number;
}> {
  const model = await getSession();

  // Tokenize (simplified — use a proper tokenizer in production)
  const inputIds = tokenize(text);
  const attentionMask = new Array(inputIds.length).fill(1);

  const feeds = {
    input_ids: new ort.Tensor("int64", BigInt64Array.from(inputIds.map(BigInt)), [1, inputIds.length]),
    attention_mask: new ort.Tensor("int64", BigInt64Array.from(attentionMask.map(BigInt)), [1, inputIds.length]),
  };

  const results = await model.run(feeds);
  const logits = results.logits.data as Float32Array;

  // Softmax
  const maxLogit = Math.max(...logits);
  const expLogits = Array.from(logits).map((l) => Math.exp(l - maxLogit));
  const sumExp = expLogits.reduce((a, b) => a + b, 0);
  const probs = expLogits.map((e) => e / sumExp);

  const maxIdx = probs.indexOf(Math.max(...probs));
  const labels = ["negative", "neutral", "positive"];

  return {
    label: labels[maxIdx],
    confidence: probs[maxIdx],
  };
}

function tokenize(text: string): number[] {
  // Simplified tokenization — use the model's tokenizer in production
  return text.split(" ").map((w) => hashWord(w) % 30000);
}

function hashWord(word: string): number {
  let hash = 0;
  for (let i = 0; i < word.length; i++) {
    hash = (hash << 5) - hash + word.charCodeAt(i);
    hash |= 0;
  }
  return Math.abs(hash);
}
```

### 10.3 Hybrid Architecture: Edge + Cloud

```
User → Edge (classification, routing) → Cloud (LLM inference)

Step 1: Edge function classifies the request (< 5ms)
  - "Is this a simple FAQ or a complex question?"
  - "Does this need a tool call?"
  - "What language is this?"

Step 2: Route based on classification
  - Simple FAQ → Cache lookup or cheap model
  - Complex question → Full LLM with RAG
  - Tool call needed → Agent with sandboxing

This hybrid approach can reduce LLM API calls by 30-50%
by handling simple cases at the edge.
```

```typescript
// app/api/chat/route.ts — hybrid edge + cloud architecture
import { classifyText } from "@/lib/edge-inference";
import { generateText, streamText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { semanticCache } from "@/lib/semantic-cache";

export const maxDuration = 60;

export async function POST(req: Request) {
  const { messages } = await req.json();
  const lastMessage = messages[messages.length - 1].content;

  // Step 1: Classify at the edge (< 5ms, free)
  const classification = await classifyText(lastMessage);

  // Step 2: Route based on classification
  if (classification.label === "faq" && classification.confidence > 0.9) {
    // Try cache first (free)
    const cached = await semanticCache.get(lastMessage);
    if (cached) {
      return Response.json({ content: cached, source: "cache" });
    }

    // Use cheap model for simple questions
    const result = await generateText({
      model: anthropic("claude-haiku-3-5-20241022"),
      messages,
    });
    await semanticCache.set(lastMessage, result.text, "claude-haiku-3-5-20241022");
    return Response.json({ content: result.text, source: "haiku" });
  }

  // Complex question — use full model with streaming
  const result = streamText({
    model: anthropic("claude-sonnet-4-20250514"),
    messages,
  });
  return result.toDataStreamResponse();
}
```

---

## 11. Cost Optimization Checklist

### 11.1 Quick Wins (< 1 day to implement)

```markdown
- [ ] Use the cheapest model that meets your quality bar (Ch 49)
      Haiku for classification, Sonnet for chat, Opus only when needed

- [ ] Enable prompt caching at the provider level
      Anthropic: automatic for repeated prefixes
      OpenAI: system message caching

- [ ] Set max_tokens on every request
      Don't let the model ramble — cap output length

- [ ] Cache deterministic responses (FAQ, classification)
      Semantic cache saves 20-40% on repeat queries

- [ ] Set spending limits at provider dashboards
      Monthly budget caps prevent surprise bills
```

### 11.2 Medium Effort (1 week)

```markdown
- [ ] Implement model routing (Ch 49)
      Route simple queries to cheap models, complex to expensive

- [ ] Add token budgets per user (Ch 54)
      Prevent any single user from burning your budget

- [ ] Batch embedding/classification requests
      Process in batches of 32+ for GPU efficiency

- [ ] Pre-compute static AI outputs
      Embeddings, summaries of documents that don't change
```

### 11.3 High Effort (1 month+)

```markdown
- [ ] Self-host a model for high-volume workloads
      Only if > 50K requests/day AND an open-source model meets quality

- [ ] Deploy quantized models
      4-bit serving at 45% lower cost with minimal quality loss

- [ ] Implement predictive auto-scaling
      Scale GPU instances based on known traffic patterns

- [ ] Edge deployment for classification/routing
      Run small models at the edge to reduce cloud LLM calls
```

---

## 12. What's Next

You have completed Part 11. You can now deploy AI applications (Ch 53), protect them with a gateway (Ch 54), sandbox dangerous agent workloads (Ch 55), automate safe deployments with CI/CD (Ch 56), and scale efficiently at the infrastructure level (Ch 57).

In Part 12, we build the **AI platform**: multi-agent orchestration (Ch 58), internal AI tools for your whole company (Ch 59), skills marketplaces for sharing capabilities (Ch 60), self-reinforcing systems that improve themselves (Ch 61), and the adoption playbook that makes your organization productive with AI (Ch 62). Everything from Parts 1-11 converges into the platform that raises the floor for everyone.

---

## Key Takeaways

1. **Most teams don't need GPU infrastructure.** If you call API providers, the infrastructure cost is a rounding error compared to token cost. Optimize application-level costs (Ch 49) first.

2. **Self-hosting only makes sense at high volume.** The break-even for Claude Sonnet vs Llama 70B is ~54,000 requests/day. Below that, the API is cheaper when you include engineering time.

3. **Scale on queue depth, not CPU.** GPU instances show low CPU utilization even when maxed out. Use request queue depth or inference latency as scaling metrics.

4. **Batching is the single biggest throughput optimization.** Processing 32 requests in one GPU batch is 20x more efficient than processing them sequentially.

5. **Quantized models are almost free performance.** 4-bit quantization reduces VRAM needs by 4x and increases throughput by 50-80% with less than 2% quality loss.

6. **Serverless inference providers (Modal, Replicate, Together) are the middle ground.** You get GPU access without managing infrastructure. Best for bursty or moderate workloads.

7. **Edge deployment works for small models only.** Classification, embeddings, and simple extraction can run at the edge. Language models need cloud or GPU infrastructure.

8. **The hybrid architecture** -- edge classification → model routing → cloud inference → CDN caching -- optimizes cost at every layer.
