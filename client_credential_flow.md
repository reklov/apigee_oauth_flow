## Flow: Client credentials

With machine-to-machine (M2M) applications, such as CLIs, daemons, or services running on your back-end, the system authenticates and authorizes the app rather than a user. For this scenario, typical authentication schemes like username + password or social logins don't make sense. Instead, M2M apps use the Client Credentials Flow (defined in [OAuth 2.0 RFC 6749, section 4.4](https://tools.ietf.org/html/rfc6749#section-4.4)), in which they pass along their Client ID and Client Secret to authenticate themselves and get a token.

### Client credentials flow without Apigee

```mermaid
sequenceDiagram

participant client as Client<br/>(Server, Cron Job, Service)
participant gate as Auth Server
participant api as Backend API

client->>gate: GET /resource <br/> (Access token)
gate->>gate: validate token
Note over gate: Token not valid
gate->>client: 401 Unauthorized (expired or invalid)

Note over client,api: Access token Creation
client->>gate: POST /token <br/>(grant_type=client_credentials, client_id, client_secret, scope)
gate->>gate: validate credentials
Note over gate: Create Token
gate->>client: return <br/> (access_token, scope, expiry)
Note over client,api: Token Use
client->>gate: GET /resource <br/>(access_token)
gate->>gate: validate token
Note over gate: Token is valid
gate->>api: GET /resource
api->>api: validate access (scope)
Note over api: Access is valid
api->>gate: return (resource)
gate->>client: return (resource)
```

### Client credentials flow with Apigee

The flow with Apigee is identical, but Apigee is taking the role of the Auth Server.

```mermaid
sequenceDiagram

participant client as Client<br/>(Server, Cron Job, Service)
participant gate as Apigee
participant api as Backend API

client->>gate: GET /resource <br/> (Access token)
gate->>gate: validate token
Note over gate: Token not valid
gate->>client: 401 Unauthorized (expired or invalid)

Note over client,api: Access token Creation
client->>gate: POST /token <br/>(grant_type=client_credentials, client_id, client_secret, scope)
gate->>gate: validate credentials
Note over gate: Create Token
gate->>client: return <br/> (access_token, scope, expiry)
Note over client,api: Token Use
client->>gate: GET /resource <br/>(access_token)
gate->>gate: validate token
Note over gate: Token is valid
gate->>api: GET /resource
api->>api: validate access (scope)
Note over api: Access is valid
api->>gate: return (resource)
gate->>client: return (resource)


```

### Client Credential flow with Apigee Edge & Envoy Proxy

Using the Envoy Proxy will remove the need to always call Apigee. In the case of a multi cloud scenario we will avvoid traffic betwenn the hyperscalers. Instead the traffic will stay inside the hyperscaler. 

```mermaid
sequenceDiagram

participant client as Client<br/>(Server, Cron Job, Service)
participant gate as Apigee
participant envoy as Envoy Sidecar
participant api as Resource API

client->>envoy: GET /resource <br/> (Access token)
envoy->>gate: validate token
Note over envoy, gate: Token not valid
envoy->>client: 401 Unauthorized (expired or invalid)

Note over client,api: Access token Creation
client->>gate: POST /token <br/>(grant_type=client_credentials, client_id, client_secret, scope)
gate->>gate: validate credentials

Note over gate: Create Token
gate->>client: return <br/> (access_token, scope, expiry)

Note over client,api: Token Use
client->>envoy: GET /resource <br/> (Access token)
envoy->>gate: validate token
Note over envoy, gate: Token is valid
envoy->>api: GET /resource
api->>api: validate access (scope)

Note over api: Access is valid
api->>envoy: return (resource)
envoy->>client: return (resource)
```


```mermaid
flowchart LR

Client -->|Token usage -- GET resource| Envoy_Proxy
Client -->|Token creation| Apigee

subgraph Resource Runtime K8S
	Envoy_Proxy -->|GET resource| Resource_API 
end

subgraph API Managment
	Envoy_Proxy -->|Token validation| Apigee
	Envoy_Proxy -->|Statistics| Apigee
end

```
