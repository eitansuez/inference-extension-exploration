# Inference extension with kgateway, using a fake LLM

This guide is similar to the lab that uses kgateway.
The difference is its use of a fake for the LLM deployment, so you can experiment with the inference extension without the cpu and memory resources needed to actually run an LLM.

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
  oci://cr.kgateway.dev/kgateway-dev/charts/kgateway-crds \
  --version v2.0.0-main \
  --namespace kgateway-system --create-namespace
```

Install kgateway:

```shell
helm upgrade --install kgateway \
  oci://cr.kgateway.dev/kgateway-dev/charts/kgateway \
  --version v2.0.0-main \
  --namespace kgateway-system --create-namespace \
  --set inferenceExtension.enabled=true
```

Above, note that the inference extension is enabled.

## Deploy a fake LLM worload

```yaml title="fakellm/fake-llm.yaml"
--8<-- "fakellm/fake-llm.yaml"
```

This fake llm is configured with the model name `fake-llm` and two alternative LoRA modules:  `fake-llm-lora-a` and `fake-llm-lora-b`.

```shell
k apply -f fakellm/fake-llm.yaml
```

## The InferencePool

```yaml title="fakellm/inferencepool.yaml"
--8<-- "fakellm/inferencepool.yaml"
```

The name of the pool is set to `fakellm-pool`.

The InferencePool resource appears to be concerned primarily with more practical aspects of how to route to the LLM workload:

1. What port does it run (`targetPortNumber`), and 
1. How do I select or identify the workload (`selector`)

```shell
k apply -f fakellm/inferencepool.yaml
```

## The InferenceModel

```yaml title="fakellm/inferencemodel.yaml"
--8<-- "fakellm/inferencemodel.yaml"
```

The model achieves the following:

1. Any requests specifying the model `fake-llm` will match this inference model
1. This model is marked with a `criticality` of `Critical`
1. This model is associated with the above inference pool, meaning that matching requests will be routed to the fake llm deployment
1. Through `targetModels` we are configuring which of the two extension models (`fake-llm-lora-a` or `fake-llm-lora-b`) to target, or what weight distributions to give to each.

```shell
k apply -f fakellm/inferencemodel.yaml
```

## Provision a Gateway

```yaml title="fakellm/gateway.yaml"
--8<-- "fakellm/gateway.yaml"
```

The configuration is straightforward: the `gatewayClassName` is set to `kgateway`.

```shell
k apply -f fakellm/gateway.yaml
```

## Configure the route

```yaml title="fakellm/route.yaml"
--8<-- "fakellm/route.yaml"
```

The route forwards requests to the inference pool directly by referencing it as a `backendRef`.

```shell
k apply -f fakellm/route.yaml
```

## Test it

Everything is now in place:  requests can be sent in to the gateway, the inference extension will be consulted, it will inspect the request, note the model `fake-llm` in the request, which will match the above InferenceModel, resulting in the request being routed to the `fake-llm-lora-a` model running as part of the cpu deployment we deployed earlier.

Capture the gateway IP address to the environment variable `GW_IP`:

```shell
export GW_IP=$(kubectl get gtw inference-gateway -o jsonpath='{.status.addresses[0].value}')
```

Send a request:

```shell
curl $GW_IP/v1/completions -H 'Content-Type: application/json' -d '{
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
      "finish_reason": "stop",
      "index": 0,
      "logprobs": null,
      "text": "This is fake LLM with LoRA-A. Greetings from module A!"
    }
  ],
  "created": 1742678062,
  "id": "fake-123",
  "model": "fake-llm-lora-a",
  "object": "text_completion",
  "usage": {
    "completion_tokens": 10,
    "prompt_tokens": 10,
    "total_tokens": 20
  }
}
```

Consider playing with the InferenceModel:

- Target the `fake-llm-lora-b` model instead.
- Make another `curl` request
- Check the response body and cnofirm that the request was handled by that model

You can also set a 50/50 weight distribution and call both models.
