---
title: AI observability
weight: 90
---

Observability helps you understand how your system is performing, identify issues, and troubleshoot problems. AI Gateway provides a rich set of observability features that help you monitor and analyze the performance of your AI Gateway and the LLM providers that it interacts with. 

## Before you begin

1. [Set up AI Gateway](/ai/tutorials/setup-gw/).

2. [Authenticate to the LLM](/ai/guides/auth/).

3. Get the external address of the gateway and save it in an environment variable.
   
   {{< tabs items="Cloud Provider LoadBalancer,Port-forward for local testing" >}}
   {{% tab %}}
   ```sh
   export INGRESS_GW_ADDRESS=$(kubectl get svc -n kgateway-system ai-gateway -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")
   echo $INGRESS_GW_ADDRESS  
   ```
   {{% /tab %}}
   {{% tab %}}
   ```sh
   kubectl port-forward deployment/ai-gateway -n kgateway-system 8080:8080
   ```
   {{% /tab %}}
   {{< /tabs >}}

## Metrics

Metrics are helpful for understanding the overall health and performance of your system. AI Gateway provides a rich set of metrics for you to monitor and analyze the performance of your AI Gateway and the LLM providers that it interacts with.

### Dynamic metadata

Dynamic metadata is a powerful feature of Envoy that allows you to attach arbitrary key-value pairs to requests and responses as they flow through the Envoy proxy. AI Gateway uses dynamic metadata to expose key metrics related to LLM provider usage. These metrics can be used to monitor and analyze the performance of your AI Gateway and the LLM providers that it interacts with.

| Dynamic Metadata Field | Description |
|-----------------------|-------------|
| `ai.gateway.kgateway.dev:total_tokens` | The total number of tokens used in the request |
| `ai.gateway.kgateway.dev:prompt_tokens` | The number of tokens used in the prompt |
| `ai.gateway.kgateway.dev:completion_tokens` | The number of tokens used in the completion |
| `envoy.ratelimit:hits_addend` | The number of tokens that were calculated to be rate limited |
| `ai.gateway.kgateway.dev:model` | The model which was specified by the user in the request |
| `ai.gateway.kgateway.dev:provider_model` | The model which the LLM provider used and returned in the response |
| `ai.gateway.kgateway.dev:provider` | The LLM provider being used, such as `OpenAI`, `Anthropic`, etc. |
| `ai.gateway.kgateway.dev:streaming` | A boolean indicating whether the request was streamed |

### Default metrics

Take a look at the default metrics that the system outputs.

1. In another tab in your terminal, port-forward the `ai-gateway` container of the gateway proxy.
   ```sh
   kubectl port-forward -n kgateway-system deploy/ai-gateway 9092
   ```

2. In the previous tab, run the following command to view the metrics.
   ```sh
   curl localhost:9092/metrics
   ```

3. In the output, search for the `ai_completion_tokens_total` and `ai_prompt_tokens_total` metrics. These metrics total the number of tokens used in the prompt and completion for the `openai` model `gpt-3.5-turbo`. 
   ```
   # HELP ai_completion_tokens_total Completion tokens
   # TYPE ai_completion_tokens_total counter
   ai_completion_tokens_total{llm="openai",model="gpt-3.5-turbo"} 539.0
   ...
   # HELP ai_prompt_tokens_total Prompt tokens
   # TYPE ai_prompt_tokens_total counter
   ai_prompt_tokens_total{llm="openai",model="gpt-3.5-turbo"} 204.0
   ```

4. Stop port-forwarding the `ai-gateway` container.

<!-- TODO: Add tracing when the HttpListenerPolicy is supported

## Tracing

Tracing helps you follow the path of a request as it is forwarded through your system. You can use tracing data to identify bottlenecks, troubleshoot problems, and optimize performance. For this tutorial, you use the all-in-one deployment of the [Jaeger](https://www.jaegertracing.io/) open source tool, which runs all components of a tracing system that you need to get started. However, because the traces are formatted for OpenTelemetry, you can configure any system that supports OTel gRPC traces.

Note that the tracing functionality of the AI Gateway integrates seamlessly with existing tracing functionality in kateway, so any existing tracing setups continue to work.

1. Install Jaeger to visualize traces.

   1. Add the Helm repo for Jaeger.
      
      ```sh
      helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
      helm repo update
      ```

   2. Deploy Jaeger into the `observability` namespace.
      
      ```yaml
      helm upgrade --install jaeger jaegertracing/jaeger \
      -n observability \
      --create-namespace \
      -f - <<EOF
      provisionDataStore:
        cassandra: false
      allInOne:
        enabled: true
      storage:
        type: memory
      agent:
        enabled: false
      collector:
        enabled: false
      query:
        enabled: false
      EOF
      ```

   3. Verify that the Jaeger all-in-one pod is running.
      
      ```sh
      kubectl get pods -n observability
      ```
      Example output:
      ```
      NAME                      READY   STATUS    RESTARTS   AGE
      jaeger-5d459f9f94-b4ckv   1/1     Running   0          26s
      ```

2. Update the `GatewayParameters` resource to configure the AI Gateway to send traces to the Jaeger collector.
   
   ```yaml
   kubectl apply -f - <<EOF
   apiVersion: gateway.kgateway.dev/v1alpha1
   kind: GatewayParameters
   metadata:
     name: kgateway-override
     namespace: kgateway-system
     labels:
       ai: tracing
   spec:
     kube:
       aiExtension:
         enabled: true
         stats:
           customLabels:
             - name: "team"
               metadataKey: "principal:team"
         tracing:
           insecure: true
           grpc:
             host: "jaeger-collector.observability.svc.cluster.local"
             port: 4317
   EOF
   ```

3. To enrich the tracing data with more details, create an additional tracing configuration for Envoy.
   
   1. Create a Backend resource that represents the Jaeger collector.
      
      ```yaml
      kubectl apply -f - <<EOF
      apiVersion: gateway.kgateway.dev/v1alpha1
      kind: Backend
      metadata:
        name: jaeger
        namespace: kgateway-system
        labels:
          ai: tracing
      spec:
        type: Static
        static:
          hosts:
            - host: jaeger-collector.observability.svc.cluster.local
              port: 4317
      EOF
      ```
   
   2. Create an `HttpListenerOption` resource that references the Gateway and Upstream. Notice that the tags use the same dynamic metadata values as the access logging and metric configurations in the previous sections.
      
      ```yaml  
      kubectl apply -f - <<EOF
      apiVersion: gateway.kgateway.dev/v1alpha1
      kind: HTTPListenerPolicy
      metadata:
        name: log-provider
        namespace: kgateway-system
        labels:
          ai: tracing
      spec:
        targetRefs:
          - group: gateway.networking.k8s.io
            kind: Gateway
            name: ai-gateway
      ...
      EOF
      ```

4. Generate traffic to the AI Gateway.
   
   {{< tabs items="Cloud Provider LoadBalancer,Port-forward for local testing" >}}
   {{% tab %}}
   ```sh
   curl "$INGRESS_GW_ADDRESS:8080/openai" -H content-type:application/json  -d '{
      "model": "gpt-3.5-turbo",
      "messages": [
        {
          "role": "system",
          "content": "You are a poetic assistant, skilled in explaining complex programming concepts with creative flair."
        },
        {
          "role": "user",
          "content": "Compose a poem that explains the concept of recursion in programming."
        }
      ]
    }'
   curl "$INGRESS_GW_ADDRESS:8080/openai" -H content-type:application/json  -d '{
      "model": "gpt-3.5-turbo",
      "messages": [
        {
          "role": "system",
          "content": "You are a poetic assistant, skilled in explaining complex programming concepts with creative flair."
        },
        {
          "role": "user",
          "content": "Shorten the recursion poem to a haiku."
        }
      ]
    }'
   {{% /tab %}}
   {{% tab %}}
   ```sh
   curl "localhost:8080/openai" -H content-type:application/json  -d '{
      "model": "gpt-3.5-turbo",
      "messages": [
        {
          "role": "system",
          "content": "You are a poetic assistant, skilled in explaining complex programming concepts with creative flair."
        },
        {
          "role": "user",
          "content": "Compose a poem that explains the concept of recursion in programming."
        }
      ]
    }'
   curl "localhost:8080/openai" -H content-type:application/json  -d '{
      "model": "gpt-3.5-turbo",
      "messages": [
        {
          "role": "system",
          "content": "You are a poetic assistant, skilled in explaining complex programming concepts with creative flair."
        },
        {
          "role": "user",
          "content": "Shorten the recursion poem to a haiku."
        }
      ]
    }'
   ```
   {{% /tab %}}
   {{< /tabs >}}

5. Review the traces in the Jaeger UI.

   1. Port-forward the Jaeger UI.

      ```sh
      kubectl port-forward svc/jaeger-query -n observability 16686:16686
      ```

   2. In your browser, navigate to [http://localhost:16686](http://localhost:16686) to view the Jaeger UI. 
   
   3. From the `Search` tab, click the **Service** dropdown and select the `gloo-ai-extension`. Configure any additional filters that you want, then click **Find Traces**. 
   
   4. Click on the trace that you want to review.
      
      TODO - add image
-->

## Next

If you haven't already, set up other features and start collecting metrics on your AI Gateway usage.

{{< cards >}}
  {{< card link="failover" title="Model failover" >}}
  {{< card link="functions" title="Function calling" >}}
  {{< card link="prompt-enrichment" title="Prompt enrichment" >}}
  {{< card link="prompt-guards" title="Prompt guards" >}}
{{< /cards >}}
