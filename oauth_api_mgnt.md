# OAuth Flow with Apigee Edge (API Management)


All flow listed here are currently only available with ApigeeEdge, running as a central entity, where every request has to run trough! Which means, that even traffic inside Azure or STACKIT needs to run via Apigee inside the Google cloud. That will raise the costs, because every request is now egress&ingress traffic from the corresponding Hyperscalers. Apigee Edge dokumentation is found here: https://docs.apigee.com/api-platform/get-started/get-started
In addition you can find the slides of the Apigee training here: [APISecurity-M1-AuthenticationAuthorizationAndOAuth](../apigateway/APISecurity-M1-AuthenticationAuthorizationAndOAuth.pdf)

##  Type of apps

##### Classify your app: public or confidential.

- Confidential apps can keep secrets confidential.

- Apps that are not confidential are considered public apps.

- Client-side web and native mobile apps are considered public apps.

##### Classify your app: trusted or untrusted. 

- First-party apps made by companies that already have full access to the user's data can be considered trusted apps.
- Third-party apps are untrusted.
- User credentials should never be entered into an unstressed app.

This will help you choose the OAuth grant type to use.

## OAuth grant types

| Grant Type                                                   | User Resources | DEPRECATED |Use Case                                                     |
| ------------------------------------------------------------ | -------------- | ------- | ------------------------------------------------------------ |
| [Client Credentials](https://oauth.net/2/grant-types/client-credentials/) | NO             | NO | System-to-system interactions<br />Resources owned by a partner, not a specific user |
| [Authorization Code](https://oauth.net/2/grant-types/authorization-code/) | YES            | NO | Requesting app is NOT trusted<br />Resources owned by the user |
| [Authorization Code - PKCE](https://oauth.net/2/pkce/) | YES            | NO | Requesting app is NOT trusted<br />Resources owned by the user <br/> PKCE (RFC 7636) is an extension to the Authorization Code flow to prevent CSRF and authorization code injection attacks. <br /><br />PKCE is not a replacement for a client secret, and PKCE is recommended even if a client is using a client secret.<br /><br /> Note: Because PKCE is not a replacement for client authentication, it does not allow treating a public client as a confidential client.|
| [Implicit](https://oauth.net/2/grant-types/implicit/)        | YES            | YES |Designed for public browser-based or mobile apps<br />**IMPLICIT NOT RECOMMENDED: USE Authorization Code instead** |
| [Password Grant](https://oauth.net/2/grant-types/password/) (Resource Owner Password) | YES | YES(*1) | Requesting app is trusted<br />Resources owned by user<br/>The Password grant type is a way to exchange a user's credentials for an access token. Because the client application has to collect the user's password and send it to the authorization server, it is not recommended that this grant be used at all anymore. And is only feasible for trusted apps.<br/>**Should only be used for server-side OAuth flow!** |

More about OAuth grants: https://alexbilbie.com/guide-to-oauth-2-grants/



## OAuth Apigee Common Pattern

All of the grant types follow a common pattern.

The app determines a need for a token, requests a new token, and finally uses the token to gain access to the protected resource.



```mermaid
sequenceDiagram

participant client as Client
participant gate as Apigee
participant api as Backend Api

client->>gate: GET /resource<br/access_token>
gate->>gate: validate token
Note over gate: Token is expired
gate->>client: 401 Unauthorized (expired or invalid)

rect rgb(191, 223, 255)
Note over client, api: Token Creation <br/>(different for each OAuth grant type)
end

client->>gate: GET /resource <br/> (access_token)
gate->>gate: validate token
Note over gate: Token is valid
gate->>api: GET /resource
api->>api: validate access (scope)
Note over api: Access is valid
api->>gate: return (resource)
gate->>client: return (resource)
```



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



#### Typical Token creation used by APIGee

![api_call_get_token](images/api_call_get_token.png)

#### Typical Token usage with APIGee

![api_token_usage](images/api_token_usage.png)

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



## Flow: [Authorization Code](https://oauth.net/2/grant-types/authorization-code/)



```mermaid
sequenceDiagram

participant user as User
participant browser as User agent <br/>(browser)
participant app as App<br/>confidential <br/>(trusted or untrusted)
participant gate as Apigee
participant auth as Auth Server<br/>(IdP)
participant api as Backend API

Note over user, api: Initiate Auth Sequence
app->>gate: GET /oauth/authorize <br/> (response_type=code, client_id, redierect_uri, scope, state)
gate->>app: return <br/>(login_page?scope={scope}&state={state}&client_id={auth_client_id})

Note over user, api: User Authentication
app->>browser:open/redirect<br/>(login_page?scope={scope}&state={state}&client_id={auth_client_id})
browser->>auth: GET login_page?scope={scope}&state={state}state&client_id={auth_client_id}
Note over browser: display login page
user->>browser: Enter login credentials
browser->>auth: submit login form
auth->>browser: return consent page
Note over browser: display consent page
user->>browser: consent
browser->>auth: submit cosent form

Note over user, api: Auth Code creation
auth->>gate: POST /oauth/authcode <br/>(client_id, redirect_uri, scope, state, user-specific info)
rect rgb(191, 223, 255)
Note over gate: validate client_id
Note over gate: validate redirect_uri<br/>against registeres redirect_uri
Note over gate: create auth code with user-specific info<br/>(custom attributes)
end
gate->>auth: return redirect_uri?code={code}&state={state}
auth->>browser: 302 redirect<br/>(redirect_uri?code={code}&state={state})
browser->>app: GET redirect_uri?code={code}&state+{state}
Note over app: extracte authcode & optional state

Note over user, api: Token Creation
app->>gate: POST /token <br/> (grant_type=authorization_code, code, redirect_uri, client_id, client_secret)
rect rgb(191, 223, 255)
Note over gate: validate client_id and client_secret
Note over gate: validate redirect_uri
Note over gate: validate auth code
Note over gate: create tokens <br/>(access token & refresh token)<br/> (automatically inherit user-specific info from auth code's custom attributes)
end
gate->>app: return <br/>(access_token, refresh_token, scope, expiry)
Note over user, api: Access Token Use
app->>gate: GET /resource<br/>(access_token)
gate->>gate: Validate Token
gate->>api: GET /resource <br/>(user info)
api->>api: validate access
api->>gate: return (resource)
gate->>app: return (resource)
```

## Flow: Authorization Code with PKCE

- Use auth code for public clients
- Public clients cannot protect secrets.
- Auth code over https is safe, but mobile redirect with auth code may be compromised. How do we keep a compromised auth code from being exchanged for an access token?
- Proof Key for Code Exchange (PKCE):
	- Specified in RFC 7636
	- Uses cryptography to guarantee that the client exchanging the auth code for tokens also initiated the auth request



The problem with public clients is that they cannot safely store secrets. Mobile apps and client side JavaScript apps are examples of public clients.

The TLS communication used for OAuth is secure over the network, but the redirect on the mobile device may not be secure.

The redirect URL contains the auth code. If the auth code is compromised, how do we keep the bad actor from exchanging the auth code for an access token if we can't require a client secret?

The answer is that we use the OAuth extension called Proof Key for Code Exchange, known as PKCE. PKCE is specified in RFC 7636.

PKCE uses cryptography to guarantee that the client exchanging an auth code for tokens is the same client that started the original auth request.

```mermaid
sequenceDiagram

participant user as User
participant browser as User agent <br/>(browser)
participant app as App<br/>confidential <br/>(trusted or untrusted)
participant gate as Apigee
participant auth as Auth Server<br/>(IdP)
participant api as Backend API

Note over user, api: Generate PKCE code_challenge and code_verifier
rect rgb(191, 223, 255)
Note over app: code_verifier=Cryptographically random num
Note over app: code_challenge=base64-urlencoded string of SHA hash of code_verifier
Note over app: store code_verifier in session
end
Note over user, api: Initiate Auth Sequence

app->>gate: GET /oauth/authorize <br/> (response_type=code, client_id, redierect_uri, scope, state)
gate->>app: return <br/>(login_page?scope={scope}&state={state}&client_id={auth_client_id})

Note over user, api: User Authentication
app->>browser:open/redirect<br/>(login_page?scope={scope}&state={state}&client_id={auth_client_id})
browser->>auth: GET login_page?scope={scope}&state={state}state&client_id={auth_client_id}
Note over browser: display login page
user->>browser: Enter login credentials
browser->>auth: submit login form
auth->>browser: return consent page
Note over browser: display consent page
user->>browser: consent
browser->>auth: submit cosent form

Note over user, api: Auth Code creation
auth->>gate: POST /oauth/authcode <br/>(client_id, redirect_uri, scope, state, user-specific info)
rect rgb(191, 223, 255)
Note over gate: validate client_id
Note over gate: validate redirect_uri<br/>against registeres redirect_uri
Note over gate: create auth code with user-specific info<br/>(custom attributes)
end
gate->>auth: return redirect_uri?code={code}&state={state}
auth->>browser: 302 redirect<br/>(redirect_uri?code={code}&state={state})
browser->>app: GET redirect_uri?code={code}&state+{state}
Note over app: extracte authcode & optional state

Note over user, api: Token Creation
app->>gate: POST /token <br/> (grant_type=authorization_code, code, redirect_uri, client_id, client_secret)
rect rgb(191, 223, 255)
Note over gate: validate client_id and client_secret
Note over gate: validate redirect_uri
Note over gate: validate auth code
Note over gate: create tokens <br/>(access token & refresh token)<br/> (automatically inherit user-specific info from auth code's custom attributes)
end
gate->>app: return <br/>(access_token, refresh_token, scope, expiry)
Note over user, api: Access Token Use
app->>gate: GET /resource<br/>(access_token)
gate->>gate: Validate Token
gate->>api: GET /resource <br/>(user info)
api->>api: validate access
api->>gate: return (resource)
gate->>app: return (resource)
```

