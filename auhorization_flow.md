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

