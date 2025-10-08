# LLM-d Inference Scheduler Performance Testing

A comprehensive performance testing framework for the [LLM-d](https://github.com/llm-d) inference scheduler using kube-burner and GuideLLM. This project automates the deployment and testing of distributed LLM inference workloads on Kubernetes/OpenShift clusters.

## Overview

This framework uses the [bring your own workload feature of kube-burner-ocp](https://kube-burner.github.io/kube-burner-ocp/latest/#custom-workload-bring-your-own-workload) to orchestrate performance tests for the LLM-d inference scheduler, which provides intelligent routing and scheduling for Large Language Model inference workloads. The tests evaluate the scheduler's performance under various load patterns and measure key Service Level Indicators (SLIs) including Time to First Token (TTFT), throughput, and resource utilization.

## Architecture

The testing framework deploys and tests the following components:

- **LLM-d Inference Scheduler**: Intelligent request router using the Gateway API Inference Extension (GAIE)
- **Endpoint Picker Plugins (EPP)**: Advanced routing algorithms for optimal endpoint selection
- **Model Serving Layer**: vLLM or simulated model servers
- **Gateway Infrastructure**: Istio-based gateway with custom routing rules
- **Load Generation**: GuideLLM-based benchmarking jobs
- **Metrics Collection**: GuideLLMPrometheus metrics exported to OpenSearch/Elasticsearch

> [!TIP]
> Resulting metrics reported by GuideLLM are parsed, normalized and indexed to the given OpenSearch/ElasticSearch endpoint thanks to the [guidellm-results-parser](https://github.com/rsevilla87/guidellm-results-parser) script.


## Prerequisites

### Required Tools

- **kube-burner-ocp**: v1.7.8+ https://github.com/kube-burner/kube-burner-ocp/releases/tag/v1.7.8
- **OpenSearch/Elasticsearch**: For metrics storage and analysis (optional but recommended)

### Cluster Requirements

Nodes with NVIDIA GPUs are required for real vLLM workloads:
  - GPU support (NVIDIA GPU Operator installed) for real vLLM workloads
  - Node Feature Discovery (NFD) for GPU node labeling
  - NVIDIA gpu operator

When using the GAIE mode:
- OSSM v3 with enabled support for Gateway API inference extension

### Environment Variables

Valid Hugging Face token 

```bash
export HF_TOKEN="your_huggingface_token"         # Required for model downloads
```

## Quick Start

### 1. Configure the Test Suite

Edit `config.yml` to customize your test parameters:

```yaml
# Key configuration variables
{{ $model := "Qwen/Qwen3-0.6B" }}         # Model to test
{{ $samples := 3 }}                       # Number of test samples
{{ $gaie := true }}                       # Enable Gateway API Inference Extension
{{ $simulator := true }}                  # Use simulator (true) or real vLLM (false)
```

GuideLLM parameters can be configured through the env variable passed to each job:

```yaml
    - objectTemplate: manifests/guidellm-job.yml
      inputVars:
        pause: 30s
        parallelism: {{ $guidellmParallelism }}
        completions: {{ $samples }}
        env:
          GUIDELLM_TARGET: {{ $guidellmTarget }}
          GUIDELLM_MAX_SECONDS: "60"
          GUIDELLM_RATE_TYPE: constant
          GUIDELLM_RATE: "20"
          GUIDELLM_DATA: "prompt_tokens=256,output_tokens=256"
        image: {{ $guidellmImage }}
        ES_SERVER: {{ $guidellmES }}
        ES_INDEX: {{ $guidellmESIndex }}
```

### 2. Running

Execute the complete test suite with kube-burner:

```bash
kube-burner-ocp init -c config.yml --es-server=${ES_SERVER} --es-index=guidellm
```

## Test Scenarios

### Steady Load Tests

Constant traffic patterns to measure baseline performance characteristics. Tests measure latency and resource utilization under sustained, predictable load.

**Service Level Indicators (SLIs):**
- P95 Time to First Token (TTFT)
- Resource usage in gateway and endpoint picker components
- Request success rate

#### Available Steady Load Tests

| Test Job | Request Rate | Prompt Tokens | Output Tokens | Use Case |
|----------|--------------|---------------|---------------|----------|
| `steady-chat-10` | 10 req/s | 256 | 256 | Interactive chat |
| `steady-chat-20` | 20 req/s | 256 | 256 | Interactive chat |
| `steady-chat-40` | 40 req/s | 256 | 256 | Interactive chat |
| `steady-rag-10` | 10 req/s | 8192 | 512 | RAG |
| `steady-rag-20` | 20 req/s | 8192 | 512 | RAG |
| `steady-rag-40` | 40 req/s | 8192 | 512 | RAG |


#### GuideLLM Test Parameters

Each test job can configure:

```yaml
env:
  GUIDELLM_TARGET: http://target-service   # Target endpoint
  GUIDELLM_MAX_SECONDS: "60"              # Test duration
  GUIDELLM_RATE_TYPE: constant            # constant, poisson, throughput
  GUIDELLM_RATE: "10"                     # Requests per second
  GUIDELLM_DATA: "prompt_tokens=256,output_tokens=256"  # Token counts
```

### Metrics Configuration:

Configured at [metrics.yml](metrics.yml), holds a series of Prometheus queries:

```yaml
- query: sum(irate(container_cpu_usage_seconds_total{...})) by (pod)
  metricName: cpu-usage

- query: sum(container_memory_working_set_bytes{...}) by (pod)
  metricName: memory-usage
```

## Analyzing Results

### Metrics in OpenSearch/Elasticsearch

The metrics indexed by the GuideLLM pods have the following structure

```yaml
[
  {
    "uuid": "c054eaf6-7b10-4dd5-a462-fbc010f7b09d",
    "job_name": "interactive-chat",
    "timestamp": "2025-09-23T23:59:06.125779",
    "strategy": "constant",
    "rate": 10.0,
    "total_requests": 609,
    "successful_requests": 586,
    "errored_requests": 0,
    "incomplete_requests": 23,
    "prompt_tokens": "128",
    "output_tokens": "128",
    "backend_model": "Qwen/Qwen3-0.6B",
    "ttft_mean_ms": 1006.2143587008272,
    "ttft_p99_ms": 1019.4225311279297,
    "itl_mean_ms": 10.423929083078537,
    "itl_p99_ms": 10.578223100797398,
    "throughput_mean_rps": 9.777764697593275,
    "throughput_p95_rps": 12.67751159149574,
    "throughput_p99_rps": 14.158848470118016,
    "request_latency_mean_seconds": 2.3300955913986363,
    "request_latency_p95_seconds": 2.3453574180603027,
    "request_latency_p99_seconds": 2.3493130207061768,
    "tokens_per_second_mean": 2426.93797445331,
    "tokens_per_second_p95": 3785.4729241877258,
    "tokens_per_second_p99": 17772.474576271186,
    "output_tokens_per_second_mean": 1251.5371956866534,
    "output_tokens_per_second_p95": 3480.7502074688796,
    "output_tokens_per_second_p99": 8272.788954635109,
    "time_per_output_token_mean_ms": 10.342492137116988,
    "time_per_output_token_p95_ms": 10.460831224918365,
    "time_per_output_token_p99_ms": 10.495580732822418
  },
  {
    "timestamp": "2025-09-23T23:59:06.126977",
    "errored": false,
    "completed": true,
    "request_latency_seconds": 2.3312907218933105,
    "tokens_per_second": 108.09462656606951,
    "output_tokens_per_second": 54.90520714467022,
    "tpot_ms": 10.28873398900032,
    "itl_ms": 10.369747642457016,
    "ttft_ms": 1014.298677444458,
    "uuid": "c054eaf6-7b10-4dd5-a462-fbc010f7b09d",
    "job_name": "interactive-chat"
  },
  {
    "timestamp": "2025-09-23T23:59:06.126454",
    "errored": false,
    "completed": true,
    "request_latency_seconds": 2.334320545196533,
    "tokens_per_second": 102.81364335067946,
    "output_tokens_per_second": 54.83394312036238,
    "tpot_ms": 10.307451710104942,
    "itl_ms": 10.388612747192383,
    "ttft_ms": 1014.9099826812744,
    "uuid": "c054eaf6-7b10-4dd5-a462-fbc010f7b09d",
    "job_name": "interactive-chat"
  }
]
```

A document with:
- The first element is the aggregate summary metrics
- Subsequent elements are timeseries entries for individual requests

```

## Project Structure

```
.
├── config.yml                    # Main kube-burner configuration
├── metrics.yml                   # Prometheus metrics queries
├── README.md                     # This file
├── infra-assets/                 # Cluster infrastructure manifests
│   ├── clusterpolicy-gpu-cluster-policy.yaml
│   ├── gateway-api.sh
│   ├── nfd-instance.yml
│   ├── nvidia-gpu-operator.yaml
│   └── ...
└── manifests/                    # Test workload manifests
    ├── guidellm-job.yml         # GuideLLM load generator
    ├── gateway.yml              # Istio Gateway
    ├── httproute.yml            # Gateway API HTTPRoute
    ├── inferencepool.yml        # InferencePool CRD
    ├── epp-deployment.yml       # Endpoint Picker Plugin
    ├── epp-config.yml           # EPP configuration
    ├── model-server-vllm-deployment.yml
    ├── model-server-sim-deployment.yml
    └── ...
```

## References

- [kube-burner Documentation](https://kube-burner.github.io/kube-burner/latest/reference/configuration/)
- [kube-burner-ocp Documentation](https://kube-burner.github.io/kube-burner-ocp/latest/)
- [LLM-d Project](https://llm-d.ai/)
- [GuideLLM](https://github.com/vllm-project/guidellm)
- [Gateway API Inference Extension](https://gateway-api-inference-extension.sigs.k8s.io/)
- [vLLM Documentation](https://docs.vllm.ai/)


## License

This project is released under the [Apache License 2.0](LICENSE).
