# Inference extension with kgateway, using a fake LLM

This guide is similar to the lab that uses kgateway.
The difference is its use of a fake for the LLM deployment, so you can experiment withe inference extension without the cpu and memory resources needed to actuall run an LLM.

Provision a Kubernetes cluster:

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

## Deploy a fake LLM worload


```yaml title="kgateway/fake-llm.yaml"
--8<-- "kgateway/fake-llm.yaml"
```

This fake llm is configured with the model name `fake-llm` and two alternative LoRA modules:  `fake-llm-lora-a` and `fake-llm-lora-b`.

```shell
k apply -f kgateway/fake-llm.yaml
```

## The InferencePool

```yaml title="kgateway/fakellm-inferencepool.yaml"
--8<-- "kgateway/fakellm-inferencepool.yaml"
```

The name of the pool is set to `fakellm-pool`.

The InferencePool resource appears to be concerned primarily with more practical aspects of how to route to the LLM workload:

1. What port does it run (`targetPortNumber`), and 
1. How do I select or identify the workload (`selector`)

```shell
k apply -f kgateway/fakellm-inferencepool.yaml
```

## The InferenceModel

```yaml title="kgateway/fakellm-inferencemodel.yaml"
--8<-- "kgateway/fakellm-inferencemodel.yaml"
```

The model achieves the following:

1. Any requests specifying the model `fake-llm` will match this inference model
1. This model is marked with a `criticality` of Critical
1. This model is associated with the above inference pool, meaning that matching requests will be routed to our cpu deployment
1. Through `targetModels` we are configuring which of the two extension models ("fake-llm-lora-a" or "fake-llm-lora-b") to target, or what weight distributions to give to each.

```shell
k apply -f kgateway/fakellm-inferencemodel.yaml
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

```yaml title="kgateway/fakellm-route.yaml"
--8<-- "kgateway/fakellm-route.yaml"
```

The route forwards requests to the inference pool directly by referencing it as a `backendRef`.

```shell
k apply -f kgateway/fakellm-route.yaml
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
  "model": "fake-llm",
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