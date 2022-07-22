
## Flow: Password Grant

Only highly-trusted applications can use the Resource Owner Password Flow (defined in [OAuth 2.0 RFC 6749, section 4.3](https://tools.ietf.org/html/rfc6749#section-4.3)), which requests that users provide credentials (username and password), typically using an interactive form. Because credentials are sent to the backend and can be stored for future use before being exchanged for an Access Token, it is imperative that the application is absolutely trusted with this information.

**The latest [OAuth 2.0 Security Best Current Practice](https://oauth.net/2/oauth-best-practice/) spec actually recommends against using the Password grant entirely, and it is being removed in the OAuth 2.1 update.**

### Password Grant flow without Apigee

```mermaid
sequenceDiagram


participant client as trusted Client
participant auth as Auth Server
participant api as Backend API

client->>api: GET /resource <br/>(access_token)
Note over api: Token is expired
api->>client: 401 Unauthorized <br/>(expired or invalid)

Note over client, api: Access and Refresh Token - Creation
client->>auth: POST /token <br/>(grant_type=password, scope, username, password)
auth->>auth: validate username, password
Note over auth: Create tokens<br/>(access & refresh token)
auth->>client: return tokens

Note over client, api: Access Token Use
client->>api: GET /resource<br/>(access_token)
api->>api: Validate Token
api->>api: validate access
api->>client: return (resource)


```

### Password Grant flow with Apigee

Orginal Apigee documentation: https://docs.apigee.com/api-platform/security/oauth/implementing-password-grant-type



```mermaid
sequenceDiagram

participant client as trusted Client
participant gate as Apigee
participant auth as Auth Server
participant api as Backend API

client->>gate: GET /resource <br/>(access_token)
Note over gate: Token is expired
gate->>client: 401 Unauthorized <br/>(expired or invalid)

Note over client, api: Access and Refresh Token -- Creation
client->>gate: POST /token <br/>(grant_type=password, client_id, client_secret, scope, username, password)
gate->>gate: validate credentials <br/> (client_id, client_secret)
gate->>auth: POST /auth <br/> (username, password)
auth->>auth: validate username, password)
auth->>gate: return <br/>(auth success, user details)
Note over gate: Create access_token and refresh_token <br/>(attaching user details)
gate->>client: return<br?>(access_token, refresh_token, scope, expiry)
Note over client, api: Access Token Use
client->>gate: GET /resource<br/>(access_token)
gate->>gate: Validate Token
gate->>api: GET /resource <br/>(user info)
api->>api: validate access
api->>gate: return (resource)
gate->>client: return (resource)

```
