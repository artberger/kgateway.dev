---
title: Retries
weight: 10
description: Specify the number of times and duration for the gateway to try a connection to an unresponsive backend service.
---

Specify the number of times and duration for the gateway to try a connection to an unresponsive backend service.

The Kubernetes Gateway API provides a way to configure retries on your HTTPRoutes. You might commonly use retries alongside [Timeouts](../timeouts/) to ensure that your apps are available even if they are temporarily unavailable.

{{% callout type="warning" %}} 
{{< reuse "docs/versions/warn-experimental.md" >}}
{{% /callout %}}

## About

A retry is the number of times a request is retried if it fails. This setting can be useful to avoid your apps from failing if they are temporarily unavailable. With retries, calls are retried a certain number of times before they are considered failed. Retries can enhance your app's availability by making sure that calls don't fail permanently because of transient problems, such as a temporarily overloaded service or network.

For more information, see the [Gateway API docs](https://gateway-api.sigs.k8s.io/geps/gep-1731/).

## Before you begin

{{< reuse "docs/snippets/prereq.md" >}}

## Step 1: Set up your environment for retries

To use retries, you need to install the experimental channel. You can also set up two things that help you test retries: a sample app that can simulate a failure and an access log policy that tracks whether the request was retried.

1. Install the experimental Kubernetes Gateway API CRDs.
   
   ```sh
   kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v{{< reuse "docs/versions/k8s-gw-version.md" >}}/experimental-install.yaml
   ```

2. Install a sample app that you can simulate a failure for, such as adding a `sleep` command to the [Bookinfo reviews app](https://istio.io/latest/docs/examples/bookinfo/).

   ```sh
   kubectl apply -f- <<EOF
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: reviews
     namespace: default
     labels:
       app: reviews
       service: reviews
   spec:
     ports:
     - port: 9080
       name: http
     selector:
       app: reviews
   ---
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: bookinfo-reviews
     namespace: default
     labels:
       account: reviews
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: reviews-v1
     namespace: default
     labels:
       app: reviews
       version: v1
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: reviews
         version: v1
     template:
       metadata:
         labels:
           app: reviews
           version: v1
       spec:
         serviceAccountName: bookinfo-reviews
         containers:
         - name: reviews
           image: docker.io/istio/examples-bookinfo-reviews-v1:1.20.3
           imagePullPolicy: IfNotPresent
           env:
           - name: LOG_DIR
             value: "/tmp/logs"
           ports:
           - containerPort: 9080
           volumeMounts:
           - name: tmp
             mountPath: /tmp
           - name: wlp-output
             mountPath: /opt/ibm/wlp/output
         volumes:
         - name: wlp-output
           emptyDir: {}
         - name: tmp
           emptyDir: {}   
   EOF
   ```

3. Apply an access log policy to the gateway that tracks the number of retries. The key log in the following example is `response_flags`, which is used to verify that the request was retried. For more information, see the [Access logging guide](../../security/access-logging/) and the [Envoy access logs response flags docs](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#response-flags).

   ```yaml
   kubectl apply -f- <<EOF
   apiVersion: gateway.kgateway.dev/v1alpha1
   kind: HTTPListenerPolicy
   metadata:
     name: access-logs
     namespace: {{< reuse "docs/snippets/namespace.md" >}}
   spec:
     targetRefs:
     - group: gateway.networking.k8s.io
       kind: Gateway
       name: http
     accessLog:
     - fileSink:
         path: /dev/stdout
         jsonFormat:
           start_time: "%START_TIME%"
           method: "%REQ(:METHOD)%"
           path: "%REQ(:PATH)%"
           response_code: "%RESPONSE_CODE%"
           response_flags: "%RESPONSE_FLAGS%"
           upstream_host: "%UPSTREAM_HOST%"
           upstream_cluster: "%UPSTREAM_CLUSTER%"
   EOF
   ```

## Step 2: Set up retries {#setup-retries}

Set up retries to the reviews app on the HTTPRoute resource.

1. Create an HTTPRoute resource that enables retries for the reviews app and sets the maximum number of retries.

   ```yaml
   kubectl apply -f- <<EOF
   apiVersion: gateway.networking.k8s.io/v1
   kind: HTTPRoute
   metadata:
     name: retry
     namespace: default
   spec:
     hostnames:
     - retry.example
     parentRefs:
     - group: gateway.networking.k8s.io
       kind: Gateway
       name: http
       namespace: {{< reuse "docs/snippets/namespace.md" >}}
     rules:
     - matches:
       - path:
           type: PathPrefix
           value: /
       backendRefs:
       - group: ""
         kind: Service
         name: reviews
         port: 9080
       retry:
         attempts: 3
         backoff: 1s
       timeouts:
         request: "20s"       
   EOF
   ```

   {{< reuse "docs/snippets/review-table.md" >}}

   | Field | Description |
   |-------|-------------|
   | hostnames | The hostnames to match the request, such as `retry.example`. |
   | parentRefs | The gateway to which the request is sent. In this example, you select the `http` gateway that you set up before you began. |
   | rules | The rules to apply to requests. |
   | matches | The path to match the request. In this example, you match any requests to the reviews app with `/`. |
   | path | The path to match the request. In this example, you match the request to the `/reviews/1` path. |
   | backendRefs | The backend service to which the request is sent. In this example, you select the `reviews` service that you set up in the previous step. |
   | retry.attempts | The number of times to retry the request. In this example, you retry the request 3 times. |
   | retry.backoff | The duration to wait before retrying the request. In this example, you wait 1 second before retrying the request. |
   | timeouts | The duration to wait before the request times out. This value is higher than the backoff value so that the request can be retried before it times out. In this example, you set the timeout to 20 seconds. |

2. Create a BackendTrafficPolicy resource to set up retry constraints for the backend service. This way, you can protect your backend from excessive retries. For more information, see the [Gateway API docs](https://gateway-api.sigs.k8s.io/api-types/backendtrafficpolicy/).

   ```yaml
   kubectl apply -f- <<EOF
   apiVersion: gateway.networking.x-k8s.io/v1alpha1
   kind: XBackendTrafficPolicy
   metadata:
     name: reviews-retry-constraint
     namespace: default
   spec:
     targetRefs:
       - group: ""
         kind: Service
         name: reviews
     retryConstraint:
       budget:
         percent: 20
         interval: 10s
       minRetryRate:
         count: 10
         interval: 1s
   EOF
   ```

   {{< reuse "docs/snippets/review-table.md" >}}

   | Field | Description |
   |-------|-------------|
   | targetRefs | The backend that this policy targets. In this example, you select the `reviews` service that you set up in the previous step. |
   | retryConstraint.budget.percent | The percentage of requests to allow to be retried. The default is 20%. |
   | retryConstraint.budget.interval | The interval to calculate the retry budget over. The default is 10 seconds. |
   | retryConstraint.minRetryRate.count | The minimum number of retries to allow per interval. The default is 10. |
   | retryConstraint.minRetryRate.interval | The interval to calculate the minimum retry rate over. The default is 1 second. |

3. Send a request to the reviews app. Verify that the request succeeds.
 
   {{< tabs items="Cloud Provider LoadBalancer,Port-forward for local testing" tabTotal="2" >}}
   {{% tab tabName="Cloud Provider LoadBalancer" %}}
   ```sh
   curl -vi http://$INGRESS_GW_ADDRESS:8080/reviews/1 -H "host: retry.example:8080"
   ```
   {{% /tab %}}
   {{% tab tabName="Port-forward for local testing" %}}
   ```sh
   curl -vi localhost:8080/reviews/1 -H "host: retry.example"
   ```
   {{% /tab %}}
   {{< /tabs >}}

   Example output for a successful response:

   ```
   HTTP/1.1 200 OK
   ...
   {"id": "1","podname": "reviews-v1-598b896c9d-l7d8l","clustername": "null","reviews": [{  "reviewer": "Reviewer1",  "text": "An extremely entertaining play by Shakespeare. The slapstick humour is refreshing!"},{  "reviewer": "Reviewer2",  "text": "Absolutely fun and entertaining. The play lacks thematic depth when compared to other plays by Shakespeare."}]}
   ```

4. Check the gateway's access logs to verify that the request was not retried.
   
   ```sh
   kubectl logs -n {{< reuse "docs/snippets/namespace.md" >}} -l gateway.networking.k8s.io/gateway-name={{< reuse "docs/snippets/default-proxy.md" >}} | tail -1
   ```

   Example output: Note that the `response_flags` field is `-`, which means that the request was not retried.

   ```json
   {
     "method": "GET",
     "path": "/reviews/1",
     "response_code": 200,
     "response_flags": "-",
     "start_time": "2025-06-16T17:24:04.268Z",
     "upstream_cluster": "kube_default_reviews_9080",
     "upstream_host": "10.244.0.24:9080"
   }
   ```

## Step 3: Trigger a retry {#trigger-retry}

Simulate a failure for the reviews app so that you can verify that the request is retried.

1. Send the reviews app to sleep, to simulate an app failure.

   ```sh
   kubectl -n default patch deploy reviews-v1 --patch '{"spec":{"template":{"spec":{"containers":[{"name":"reviews","command":["sleep","20h"]}]}}}}'
   ```

2. Send another request to the reviews app. This time, the request fails.
   
   {{< tabs items="Cloud Provider LoadBalancer,Port-forward for local testing" tabTotal="2" >}}
   {{% tab tabName="Cloud Provider LoadBalancer" %}}
   ```sh
   curl -vi http://$INGRESS_GW_ADDRESS:8080/reviews/1 -H "host: retry.example:80"
   ```
   {{% /tab %}}
   {{% tab tabName="Port-forward for local testing" %}}
   ```sh
   curl -vi localhost:8080/reviews/1 -H "host: retry.example"
   ```
   {{% /tab %}}
   {{< /tabs >}}

   Example output:

   ```
   HTTP/1.1 503 Service Unavailable
   ...
   upstream connect error or disconnect/reset before headers. retried and the latest reset reason: remote connection failure, transport failure reason: delayed connect error: Connection refused
   ```

3. Check the gateway's access logs to verify that the request was retried.
   
   ```sh
   kubectl logs -n {{< reuse "docs/snippets/namespace.md" >}} -l gateway.networking.k8s.io/gateway-name={{< reuse "docs/snippets/default-proxy.md" >}} | tail -1 | jq
   ```

   Example output: Note that the `response_flags` field now has values as follows:

   * `URX` means `UpstreamRetryLimitExceeded`, which verifies that the request was retried.
   * `UF` means `UpstreamOverflow`, which verifies that the request failed.
   
   ```json
   {
     "method": "GET",
     "path": "/reviews/1",
     "response_code": 503,
     "response_flags": "URX,UF",
     "start_time": "2025-06-16T17:26:07.287Z",
     "upstream_cluster": "kube_default_reviews_9080",
     "upstream_host": "10.244.0.25:9080"
   }
   ```

## Cleanup

{{< reuse "docs/snippets/cleanup.md" >}}

1. Delete the HTTPRoute and XBackendTrafficPolicy resources that configure your retries.
   
   ```sh
   kubectl delete httproute retry -n default
   kubectl delete xbackendtrafficpolicy reviews-retry-constraint -n default
   ```

2. Delete the reviews app.

   ```sh
   kubectl delete deploy reviews-v1 -n default
   kubectl delete svc reviews -n default
   kubectl delete sa bookinfo-reviews -n default
   ```

3. Delete the access log policy.

   ```sh
   kubectl delete httplistenerpolicy access-logs -n {{< reuse "docs/snippets/namespace.md" >}}
   ```
