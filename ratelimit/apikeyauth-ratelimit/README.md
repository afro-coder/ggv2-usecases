
### Ratelimiting per API-Key in Gloo Gateway V2

The concept is ported over from the Gloo Edge v1 install using Portal.

The API Key has a plan associated with it, ideally per-requests.


To begin with we create secrets as such, please note the type and the data associated with it.


```
---
apiVersion: v1
kind: Secret
metadata:
  name: team-a-apikey-1
  namespace: httpbin
  labels:
    team: team-a
    plan: team-a-3-tps
type: extauth.solo.io/apikey
stringData:
  api-key: N2YwMDIxZTEtNGUzNS1jNzgzLTRkYjAtYjE2YzRkZGVmNjcy
  plan: team-a-3-tps
  username: demo-user
---
apiVersion: v1
kind: Secret
metadata:
  name: team-b-apikey-2
  namespace: httpbin
  labels:
    team: team-a
    plan: team-a-5-tps
type: extauth.solo.io/apikey
stringData:
  api-key: N2YwMDIxZTEtNGUzNS1jNzgzLTRkYjAtYjE2YzRkZGVmNjcz
  plan: team-a-5-tps
  username: demo-user
```

To create an API-key auth we need to add the following authconfig

```
apiVersion: extauth.solo.io/v1
kind: AuthConfig
metadata:
  name: apikey-auth
  namespace: httpbin
spec:
  configs:

    - apiKeyAuth:
        # The request header name that holds the API key.
        # This field is optional and defaults to api-key if not present.
        headerName: api-key
        headersFromMetadataEntry:
          x-solo-username:
            name: username
            required: false
          x-solo-plan:
            name: plan
            required: false
        labelSelector:
          team: team-a
```

Then we create the Ratelimit for this

```
apiVersion: ratelimit.solo.io/v1alpha1
kind: RateLimitConfig
metadata:
  name: ratelimit-config
  namespace: httpbin
spec:
  raw:
    descriptors:
      - descriptors:
        - key: userId
          rateLimit:
            requestsPerUnit: 10
            unit: MINUTE
        key: usagePlan
        value: defaultPlan
      - descriptors:
        - key: userId
          rateLimit:
            requestsPerUnit: 5
            unit: MINUTE
        key: usagePlan
        value: team-a-5-tps
      - descriptors:
        - key: userId
          rateLimit:
            requestsPerUnit: 3
            unit: MINUTE
        key: usagePlan
        value: team-a-3-tps
    rateLimits:
      - actions:
        - requestHeaders:
            descriptorKey: usagePlan
            headerName: x-solo-plan
        - metadata:
            descriptorKey: userId
            metadataKey:
              key: envoy.filters.http.ext_authz
              path:
                - key: userId
```

Finally we create the `GlooTrafficPolicy`

```
apiVersion: gloo.solo.io/v1alpha1
kind: GlooTrafficPolicy
metadata:
  name: test-extauth-policy
  namespace: httpbin
spec:
  targetRefs:
    - name: mycustom-gateway
      group: gateway.networking.k8s.io
      kind: Gateway
  glooExtAuth:
    authConfigRef:
      name: apikey-auth
      namespace: httpbin
  glooRateLimit:
    global:
      rateLimitConfigRefs:
        - name: ratelimit-config
          namespace: httpbin

```
