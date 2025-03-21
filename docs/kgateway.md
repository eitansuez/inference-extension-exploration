# Using the GW API Inference extension with kgateway

This guide is similar to the GW API inference extension guide that uses Envoy Gateway.
This guide focuses on the same scenario, but running with [kgateway](https://kgateway.dev/) instead.

The idea is to setup in a Kubernetes cluster a mechanism to route traffic to backend LLMs.

## Prerequisites

- amd64 CPU architecture (required by vLLM).
- Lots of CPU and RAM.
- Tools:  kubectl, helm

## A k8s cluster

I chose k3d:

```shell
k3d cluster create my-k8s-cluster \
    --api-port 6443 \
    --k3s-arg "--disable=traefik@server:0" \
    --port 80:80@loadbalancer \
    --port 443:443@loadbalancer
```

## Install kgateway

Install the GW API CRDs:

```shell
kubectl apply -f \
  https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/experimental-install.yaml
```

The inference extension CRDs also need to be installed:

```shell
k apply -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/v0.2.0/manifests.yaml
```

Install the kgateway CRDs:

```shell
helm upgrade --install kgateway-crds \
  https://github.com/danehans/toolbox/raw/refs/heads/main/charts/8836480ba3-kgateway-crds-1.0.1-dev.tgz \
  --version v2.0.0-main \
  --namespace kgateway-system --create-namespace
```

Install kgateway:

```shell
helm upgrade --install kgateway \
  https://github.com/danehans/toolbox/raw/refs/heads/main/charts/8836480ba3-kgateway-1.0.1-dev.tgz \
  --version v2.0.0-main \
  --namespace kgateway-system --create-namespace \
  --set inferenceExtension.enabled=true \
  --set image.registry=danehans
```

Above, note that the inference extension is enabled.

## Deploy an LLM worload

The guide offers two options.  We opt for the cpu deployment option.

```yaml title="kgateway/cpu-deployment.yaml"
--8<-- "kgateway/cpu-deployment.yaml"
```

The cpu deployment option uses a relatively small model:  [`Qwen/Qwen2.5-1.5B-Instruct`](https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct).
This model is not gated and so does not require a Hugging Face token.

The container requires 12 cpus and 9Gi of memory (see the `resources` section).

The perhaps important thing to note about the running container is that it supposedly configures two models, `tweet-summary-0` and `tweet-summary-1` through lora adapters that extend the base model.

The default deployment sets 3 replicas which requires a lot of resources.
So I set my manifest to use just one replica.

```shell
k apply -f kgateway/cpu-deployment.yaml
```

Allow 5-10 minutes before the pod is up and running.

## The InferencePool

```yaml title="kgateway/inference-pool.yaml"
--8<-- "kgateway/inference-pool.yaml"
```

The name of the pool is set to `qwen-pool`.

The InferencePool resource appears to be concerned primarily with more practical aspects of how to route to the LLM workload:

1. What port does it run (`targetPortNumber`), and 
1. How do I select or identify the workload (`selector`)

```shell
k apply -f kgateway/inference-pool.yaml
```

## The InferenceModel

```yaml title="kgateway/inference-model.yaml"
--8<-- "kgateway/inference-model.yaml"
```

The model achieves the following:

1. Any requests specifying the model "tweet-summary" will match this inference model
1. This model is marked with a `criticality` of Critical
1. This model is associated with the above inference pool, meaning that matching requests will be routed to our cpu deployment
1. Through `targetModels` we are configuring which of the two extension models ("tweet-summary-0" or "tweet-summary-1") to target, or what weight distributions to give to each.

```shell
k apply -f kgateway/inference-model.yaml
```

## Provision a Gateway

```yaml title="kgateway/gateway.yaml"
--8<-- "kgateway/gateway.yaml"
```

The configuration is straightforward: the `gatewayClassName` is set to `kgateway`.

It's curious that two listeners are configured, presumably to demonstrate that you can segregate LLM requests from non-LLM requests.  In other words, dedicate a listener for LLM requests named `llm-gw`.

The inference extension is already hooked up to the gateway.

```shell
k apply -f kgateway/gateway.yaml
```

## Configure the route

```yaml title="kgateway/route.yaml"
--8<-- "kgateway/route.yaml"
```

The route forwards requests to the inference pool directly by referencing it as a `backendRef`.

```shell
k apply -f kgateway/route.yaml
```

## Test it

Everything is now in place:  requests can be sent in to the gateway, the inference extension will be consulted, it will inspect the request, note the model `tweet-summary` in the request, which will match the above InferenceModel, resulting in the request being routed to the `tweet-summary-1` model running as part of the cpu deployment we deployed earlier.

Capture the gateway IP address to the environment variable `GW_IP`:

```shell
export GW_IP=$(kubectl get gtw inference-gateway -o jsonpath='{.status.addresses[0].value}')
```

Send a request:

```shell
curl $GW_IP:8081/v1/completions -H 'Content-Type: application/json' -d '{
  "model": "tweet-summary",
  "prompt": "Write as if you were a critic: San Francisco",
  "max_tokens": 100,
  "temperature": 0
}' | jq
```

Here is a sample response:

```json
{
  "choices": [
    {
      "finish_reason": "length",
      "index": 0,
      "logprobs": null,
      "prompt_logprobs": null,
      "stop_reason": null,
      "text": " Giants - 2019\n\nThe San Francisco Giants have been one of the most successful teams in Major League Baseball over the past few years, and they continue to be a force to be reckoned with. The team has won three World Series championships in the last five seasons, including their first title since 1954.\n\nIn 2019, the Giants continued their dominance by winning the National League West divisional title for the third time in four years. They finished the"
    }
  ],
  "created": 1742577733,
  "id": "cmpl-30ea8f02-2aa7-4648-bb28-efe88c43517c",
  "model": "tweet-summary-1",
  "object": "text_completion",
  "usage": {
    "completion_tokens": 100,
    "prompt_tokens": 10,
    "prompt_tokens_details": null,
    "total_tokens": 110
  }
}
```

Consider playing with the InferenceModel:

- Target the other model instead.
- Make another curl request
- Verify in the response that the request was handled by that model

## Thoughts

This is much better than the previous guide.
The extension is still there but no longer needs to be managed explicitly.

As I review this model diagram:

![](images/inference-overview.svg)

The configuration matches this picture perfectly:  the route uses the InferencePool as a `backendRef`, which in turn acts as a proxy for the backing LLM workload.

This redresses the issues I cited from the previous guide.