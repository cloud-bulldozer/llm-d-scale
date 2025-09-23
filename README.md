# LLM-d inference scheduler performance

## Scenarios

### Steady load

Constant traffic to measure baseline latency, One request after another

Routing SLIs: P95 TTFT
Hardware SLIs: resource usage in the gateway and endpoint picker components

### Throughput

Run as many requests as possible in parallel to measure maximum throughput. To be executed with different concurrency levels

Routing SLIs: Throughput, P95 TTFT
Hardware SLIs: resource usage in the gateway and endpoint picker components


### Sweep

Emulate an scenario with a irregular request distribution, where a big amount of events happen in a fixed interval of time

Routing SLIs: Throughput, P95 TTFT
Hardware SLIs: resource usage in the gateway and endpoint picker components


### Tooling

GuideLLM


## Bonus track: Endpoint picker plugins


EPP Metrics -> https://github.com/kubernetes-sigs/gateway-api-inference-extension/blob/main/pkg/epp/server/runserver.go

Endpoint picker plugins
- kv-cache-scorer:: Sores the candidate pdos on their KV cache utilization
- precise-prefix-cache-scorer: When enabled, the scorer will use the llm-d-kv-cache-manager to track the KV-cache states across the vLLM instances. (scores a request based on KV-cache localities)
  - indexerConfig: Configuration for the kvcache.Indexer.
  - kvEventsConfig: Configuration for the kvevents.Pool.
  - Configuation of both components is: https://github.com/llm-d/llm-d-kv-cache-manager/blob/main/docs/configuration.md
- prefix-cache-scorer: (takes advantage of the prefix caching, i.e vllm APC, automatic prefix cachign) Scores pods based on the amount of the prompt is believed to be in the pod's KvCache.
  - blockSize: This is the size of each block in number of bytes. vLLM default block size is 16 tokens. Assume 4 characters per token, the default is set to 64 in EPP. The default is recommended unless performance is critical for use cases with extremely long inputs.
  - maxPrefixBlocksToMatch: The maximum number of blocks to find prefix match
  - lruCapacityPerServer: Maximum capacity the prefix LRU cache in number of block hashes per server (pod).
- max-score-picker: Pics the pods with higher score
- single-profile-handler: A Profile Handler must be specified, unless the configuration only contains one profile, in which case the SingleProfileHandler will be used.
- active-request-score: Scores pods based on the number of active requests being served per pod. 
- load-aware-scorer: Scores pods based on their load, based on the number of requests concurrently being processed. A threshold is provided which is used to determine what is considered an overloaded pod.
- max-score-picker: Picks pods with the max score from the list of candidates
- prefill-header-handler: Sets a header for use in disaggregated prefill/decode
- dedode-filter: Filters out pods that are not marked either as decode or both prefill and decode. The filter looks for the label llm-d.ai/role, with a value of either `decode` or `both`.
- prefill-filter: Filters out pods that are not marked as prefill. The filter looks for the label `llm-d.ai/role`, with a value of `prefill`
