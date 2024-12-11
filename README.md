# Gloo AI Gateway Auth and Rate Limit Demo

![ai-gateway-main](images/ai-gateway-main.svg)

### Setup the Environment

Add repo
```bash
helm repo add gloo-ee-helm https://storage.googleapis.com/gloo-ee-helm
helm repo update
```

Install gateway api crds
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
```

Install gloo gateway
```bash
helm upgrade --install -n gloo-system \
gloo-gateway gloo-ee-helm/gloo-ee \
--create-namespace \
--version 1.18.0-rc1 \
--set-string license_key=$GLOO_LICENSE_KEY \
-f -<<EOF
gloo:
  kubeGateway:
    enabled: true
  gatewayProxies:
    gatewayProxy:
      disabled: true
  discovery:
    enabled: false
gloo-fed:
  enabled: false
  glooFedApiserver:
    enable: false
# disable everything else for a simple deployment
observability:
  enabled: false
prometheus:
  enabled: false
grafana:
  defaultInstallationEnabled: false
gateway-portal-web-server:
  enabled: false
EOF
```

check to see that gloo gateway and its components have been created
```bash
kubectl get pods -n gloo-system 
```

Install ai gateway
```bash
kubectl apply -f aig-base.yaml
```

check to see that ai gateway proxy has been created
```bash
kubectl get pods -n gloo-system | grep gloo-proxy-ai-gateway
```

Install ollama deployment which will load qwen 0.5b and 1.8b models
```bash
kubectl apply -f ollama-deploy
```

check to see that the `ollama-qwen` deployment has been created
```bash
kubectl get pods -n ollama
```

## Configure the Rate Limit Demo

Configure a simple route to qwen-0.5b LLM backend
```bash
kubectl apply -f route
```

Get ai gateway LB address
```bash
export GATEWAY_IP=$(kubectl get svc -n gloo-system gloo-proxy-ai-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}{.status.loadBalancer.ingress[0].hostname}')
echo "Gateway IP: $GATEWAY_IP"
```

Curl ai gateway endpoint for qwen model served by ollama (note no model defined)
```bash
curl http://$GATEWAY_IP:8080/qwen   -H "Content-Type: application/json" -d '{
    "messages": [
      {
        "role": "system",
        "content": "You are a solutions architect for kubernetes networking, skilled in explaining complex technical concepts surrounding API Gateway, Service Mesh, and CNI"
      },
      {
        "role": "user",
        "content": "Write me a 20 word pitch on why I should use a service mesh in my kubernetes cluster"
      }
    ]
  }'
```

Responses should come from `qwen-0.5b`

```bash
{"id": "chatcmpl-808", "object": "chat.completion", "created": 1730758807, "model": "qwen:0.5b", "system_fingerprint": "fp_ollama", "choices": [{"index": 0, "message": {"role": "assistant", "content": "Use service mesh to orchestrate networking between multiple applications.\u63d0\u9ad8\u4e86 application security and availability. Sign up and start with service mesh."}, "finish_reason": "stop"}], "usage": {"prompt_tokens": 59, "completion_tokens": 26, "total_tokens": 85}}
```

## Set up Access Control

Take a look at the VirtualHostOption where we will configure JWT based access control
```bash
cat access-control/jwt-provider.yaml
```

Configure a VirtualHostOption that ensures that only authenticated and authorized users can access the LLM APIs
```bash
kubectl apply -f access-control
```

Curl ai gateway endpoint for qwen model served by ollama, note that the request returns a 401 HTTP response code and the message indicating that the `Jwt is missing`
```bash
curl -v http://$GATEWAY_IP:8080/qwen   -H "Content-Type: application/json" -d '{
    "messages": [
      {
        "role": "system",
        "content": "You are a solutions architect for kubernetes networking, skilled in explaining complex technical concepts surrounding API Gateway, Service Mesh, and CNI"
      },
      {
        "role": "user",
        "content": "Write me a 20 word pitch on why I should use a service mesh in my kubernetes cluster"
      }
    ]
  }'
```

Save the following JWT tokens as environment variables
```bash
export ALICE_TOKEN=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyAiaXNzIjogInNvbG8uaW8iLCAib3JnIjogInNvbG8uaW8iLCAic3ViIjogImFsaWNlIiwgInRlYW0iOiAiZGV2IiwgImxsbXMiOiB7ICJvcGVuYWkiOiBbICJncHQtMy41LXR1cmJvIiBdIH0gfQ.I7whTti0aDKxlILc5uLK9oo6TljGS6JUrjPVd6z1PxzucUa_cnuKkY0qj_wrkzyVN5djy4t2ggE1uBO8Llpwi-Ygru9hM84-1m53aO07JYFya1VTDsI25tCRG8rYhShDdAP5L935SIARta2QtHhrVcd1Ae7yfTDZ8G1DXLtjR2QelszCd2R8PioCQmqJ8PeKg4sURhu05GlBCZoXES9-rtPVbe6j3YLBTodJAvLHhyy3LgV_QbN7IiZ5qEywdKHoEF4D4aCUf_LqPp4NoqHXnGT4jLzWJEtZXHQ4sgRy_5T93NOLzWLdIjgMjGO_F0aVLwBzU-phykOVfcBPaMvetg

export BOB_TOKEN=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyAiaXNzIjogInNvbG8uaW8iLCAib3JnIjogInNvbG8uaW8iLCAic3ViIjogImJvYiIsICJ0ZWFtIjogIm9wcyIsICJsbG1zIjogeyAibWlzdHJhbGFpIjogWyAibWlzdHJhbC1sYXJnZS1sYXRlc3QiIF0gfSB9.p7J2UFwnUJ6C7eXsFCSKb5b7ecWZ75JO4TUJHafjLv8jJ7GzKfJVk7ney19PYUrWrO4ntwnnK5_sY7yaLUBCJ3fv9pcoKyRtJTw1VMMTQsKkWFgvy-jEwc9M-D5lrUfR1HXGEUm6NBaj_Ja78XScPZb_-APPqMIvzDZU04vd6hna3UMc4DZE0wcnTjOqoND0GllHLupYTfgX0v9_AYJiKRAcJvol1W14dI7szpY5GFZtPqq0kl1g0sJPg-HQKwf7Cfvr_JLjkepNJ6A1lsrG8QbuUvMUAdaHzwLvF3L_G6VRjEte6okZpaq0g2urWpZgdNmPVN71Q_0WhyrJTr6SyQ
```

As described in the docs, here are the claims for Alice and Bob respectively
```bash
{
  "iss": "solo.io",
  "org": "solo.io",
  "sub": "alice",
  "team": "dev",
  "llms": {
    "openai": [
      "gpt-3.5-turbo"
    ]
  }
}

{
  "iss": "solo.io",
  "org": "solo.io",
  "sub": "bob",
  "team": "ops",
  "llms": {
    "mistralai": [
      "mistral-large-latest"
    ]
  }
}
```

Repeat the curl command but this time include the JWT token for Alice in the `Authorization` header
```bash
curl -v http://$GATEWAY_IP:8080/qwen -H "Authorization: Bearer $ALICE_TOKEN" -H "Content-Type: application/json" -d '{
    "messages": [
      {
        "role": "system",
        "content": "You are a solutions architect for kubernetes networking, skilled in explaining complex technical concepts surrounding API Gateway, Service Mesh, and CNI"
      },
      {
        "role": "user",
        "content": "Write me a 20 word pitch on why I should use a service mesh in my kubernetes cluster"
      }
    ]
  }'
```

Try Bob's JWT token to verify that the request succeeds as well
```bash
curl -v http://$GATEWAY_IP:8080/qwen -H "Authorization: Bearer $BOB_TOKEN" -H "Content-Type: application/json" -d '{
    "messages": [
      {
        "role": "system",
        "content": "You are a solutions architect for kubernetes networking, skilled in explaining complex technical concepts surrounding API Gateway, Service Mesh, and CNI"
      },
      {
        "role": "user",
        "content": "Write me a 20 word pitch on why I should use a service mesh in my kubernetes cluster"
      }
    ]
  }'
```

We can further scope down and control access to a specific set of claims in the JWT to just the `solo.io` org and `dev` team
```bash
cat access-control/rbac/rbac-route-option.yaml
```

Apply the policy
```bash
kubectl apply -f access-control/rbac
```

Try Alice's JWT token, the first request should succeed because she is part of the dev team
```bash
curl -v http://$GATEWAY_IP:8080/qwen -H "Authorization: Bearer $ALICE_TOKEN" -H "Content-Type: application/json" -d '{
    "messages": [
      {
        "role": "system",
        "content": "You are a solutions architect for kubernetes networking, skilled in explaining complex technical concepts surrounding API Gateway, Service Mesh, and CNI"
      },
      {
        "role": "user",
        "content": "Write me a 20 word pitch on why I should use a service mesh in my kubernetes cluster"
      }
    ]
  }'
```

Try Bob's token, this will return a `RBAC: access denied` because he is part of the ops team
```bash
curl -v http://$GATEWAY_IP:8080/qwen -H "Authorization: Bearer $BOB_TOKEN" -H "Content-Type: application/json" -d '{
    "messages": [
      {
        "role": "system",
        "content": "You are a solutions architect for kubernetes networking, skilled in explaining complex technical concepts surrounding API Gateway, Service Mesh, and CNI"
      },
      {
        "role": "user",
        "content": "Write me a 20 word pitch on why I should use a service mesh in my kubernetes cluster"
      }
    ]
  }'
```

Let's remove the policy before the next example
```bash
kubectl delete -f access-control/rbac
```

## Tiered Rate Limiting

The following policy allows us to rate limit requests based on token count instead of request count, which aligns more closely to how LLMs are consumed. Additionally we can apply multiple policies to the same route to enforce behavior

Here we are enforcing the following:
- rate limit of 150 tokens per MINUTE.
- rate limit of 300 tokens per HOUR.
- rate limit of 800 tokens per DAY.

```bash
cat tiered-rate-limit/per-user-counter-minute-rlc.yaml
cat tiered-rate-limit/per-user-counter-hour-rlc.yaml
cat tiered-rate-limit/per-user-counter-day-rlc.yaml
```

We will then configure a route option that will apply these three policies to the `ollama-qwen` HTTPRoute using a `targetRef`
```bash
cat tiered-rate-limit/rlc-route-option.yaml
```

Create the policies and route option
```bash
kubectl apply -f tiered-rate-limit
```

Try Alice's JWT token, the first request should succeed
```bash
curl -v http://$GATEWAY_IP:8080/qwen -H "Authorization: Bearer $ALICE_TOKEN" -H "Content-Type: application/json" -d '{
    "messages": [
      {
        "role": "system",
        "content": "You are a solutions architect for kubernetes networking, skilled in explaining complex technical concepts surrounding API Gateway, Service Mesh, and CNI"
      },
      {
        "role": "user",
        "content": "Write me a 20 word pitch on why I should use a service mesh in my kubernetes cluster"
      }
    ]
  }'
```

Try it a few more times and the requests will be rate limited once the quota of 150 tokens are counted in a MINUTE
```bash
< HTTP/1.1 429 Too Many Requests
< x-envoy-ratelimited: true
< date: Thu, 21 Nov 2024 22:58:21 GMT
< server: envoy
< content-length: 0
```

Try Bob's JWT token, the request should succeed since he has a different user-id, but the subsequent requests will also be rate limited
```bash
curl -v http://$GATEWAY_IP:8080/qwen -H "Authorization: Bearer $BOB_TOKEN" -H "Content-Type: application/json" -d '{
    "messages": [
      {
        "role": "system",
        "content": "You are a solutions architect for kubernetes networking, skilled in explaining complex technical concepts surrounding API Gateway, Service Mesh, and CNI"
      },
      {
        "role": "user",
        "content": "Write me a 20 word pitch on why I should use a service mesh in my kubernetes cluster"
      }
    ]
  }'
```

Note that if you pause for 1 minute, you will be able to make requests to the backend LLMs again. After a couple more requests you should notice that you are rate limited again, but this time because of the rate limit of 300 requests per HOUR. You can confirm this by waiting another minute, and demonstrating that you have also hit the hourly token limit

## Cleanup

Remove routes and policies
```bash
kubectl delete -f tiered-rate-limit
kubectl delete -f access-control/rbac
kubectl delete -f access-control
kubectl delete -f route
```

Delete ollama deployment
```bash
kubectl delete -f ollama-deploy
```

Delete AI Gateway
```bash
kubectl delete -f aig-base.yaml
```

Uninstall Gloo
```bash
helm uninstall gloo-gateway -n gloo-system
```

## AI gateway metrics

Port forward to 19000 of the ai gateway pod to view metrics
```bash
kubectl port-forward -n gloo-system $(kubectl get pod -l gateway.networking.k8s.io/gateway-name=ai-gateway -n gloo-system -o jsonpath='{.items[0].metadata.name}') 19000:19000
```

