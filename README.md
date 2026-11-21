# dvra-damn-vulnerable-restaurant-api-solutions-guide
Complete DVRA (Damn Vulnerable Restaurant API) solutions and walkthroughs for every level. Covers BOLA, BFLA, BOPLA, SSRF, JWT forgery, mass assignment, privilege escalation, and RCE, with causes, processes, and exact payloads for each API vuln.

# DVRA Solutions: Complete Walkthroughs for the RESTaurant API Game

This repository is a complete, structured guide to solving the Damn Vulnerable RESTaurant API Game (DVRA). It covers the full attack chain from information disclosure to remote code execution, mapped to the OWASP API Security Top 10. Every entry explains the vulnerability, the level where it appears, the cause, the exploitation process, and the exact request or payload used. All targets assume the API is reachable at `http://localhost:8091` or `http://localhost:8080`, depending on your deployment.

## Contents

- [Setup and Recon](#setup-and-recon)
- [Level 0: Information Disclosure](#level-0-information-disclosure)
- [Level 1: Unrestricted Menu Item Deletion (BFLA)](#level-1-unrestricted-menu-item-deletion-bfla)
- [Level 2: BOLA (Broken Object Level Authorization)](#level-2-bola-broken-object-level-authorization)
- [Level 3: Privilege Escalation via BOPLA](#level-3-privilege-escalation-via-bopla)
- [Level 4: Server Side Request Forgery (SSRF)](#level-4-server-side-request-forgery-ssrf)
- [Level 5: Remote Code Execution (RCE)](#level-5-remote-code-execution-rce)
- [Broken Authentication: JWT Forgery](#broken-authentication-jwt-forgery)
- [Unrestricted Resource Consumption](#unrestricted-resource-consumption)
- [Broken Function Level Authorization (BFLA)](#broken-function-level-authorization-bfla)

## Setup and Recon

DVRA runs with Docker and exposes an OpenAPI specification that lists every endpoint, method, and parameter. This specification is the starting point for all reconnaissance.

Start the application with Docker Compose, then open the Swagger interface at `http://localhost:8091/docs` or the OpenAPI JSON at `http://localhost:8091/openapi.json`. Import the JSON into Postman or Burp Suite to map the full attack surface[reference:2].

Register two accounts before testing. One account acts as the attacker, the other as the victim. Obtain a JWT for each account by sending credentials to the `POST /token` endpoint.

The token endpoint uses form data, not JSON. The returned access_token is a JWT that must be sent as a Bearer token in subsequent requests.

Level 0: Information Disclosure
Vuln: Information Disclosure

Level: 0

Cause: The /healthcheck endpoint returns server and framework details in HTTP headers, including the Python version and FastAPI version.

Process: Send a GET request to /healthcheck and inspect the response headers. The disclosed stack information helps plan framework specific attacks.

Payload: GET /healthcheck HTTP/1.1 against the target. The response header x-powered-by reveals Python 3.10 and FastAPI ^0.103.0.

Level 1: Unrestricted Menu Item Deletion (BFLA)
Vuln: Broken Function Level Authorization (BFLA)

Level: 1

Cause: The DELETE /menu/{id} endpoint does not check the caller's role. A low privileged Customer account can delete menu items, which should be a Chef or admin function.

Process: Obtain a token for a normal Customer account, list the menu with GET /menu, then send a DELETE request for a chosen item ID using the Customer token.

Payload: DELETE /menu/17 HTTP/1.1 with Authorization: Bearer <customer_token>. The item is removed, confirming the missing function level authorization check.

Level 2: BOLA (Broken Object Level Authorization)
Vuln: BOLA / IDOR

Level: 2

Cause: The PUT /profile endpoint authenticates the user but does not verify that the username in the request body matches the identity in the JWT. The endpoint trusts the body value.

Process: Register two accounts, userA and userB. Authenticate as userA, then send a profile update request with username set to userB. The API updates userB's profile even though the request carries userA's token.

Payload: A PUT /profile request with userA's Bearer token and a JSON body containing "username": "userB" alongside the fields to change. The response confirms the victim's profile was modified. The same flaw exists on the GET /orders/{id} endpoint, where order IDs are sequential integers and can be enumerated to read other users' orders.

Level 3: Privilege Escalation via BOPLA
Vuln: Broken Object Property Level Authorization (BOPLA) / Mass Assignment

Level: 3

Cause: The PATCH /profile endpoint accepts properties that are not part of the intended request schema. The role property is not listed in the API documentation, but the API binds it if present.

Process: Send a profile update that includes "role": "Chef" in the JSON body. The API applies the new role, granting elevated privileges.

Payload: A PATCH /profile request with a valid token and a body such as {"role": "Chef"}. After the update, verify the new role with GET /profile and access Chef only endpoints.

Level 4: Server Side Request Forgery (SSRF)
Vuln: Server Side Request Forgery

Level: 4

Cause: The PUT /menu endpoint accepts an image_URL property. The server fetches the supplied URL and returns the content as a base64 encoded string. There is no validation or allowlist for the destination.

Process: Supply an external URL under your control to confirm the server makes the request. Then replace it with an internal URL that reaches a privileged service, such as the Chef password reset endpoint on localhost.

Payload: A PUT /menu request with "image_URL": "http://webhook.site/<id>" returns the fetched content in base64. Replace the URL with "http://localhost:8091/admin/reset-chef-password" to trigger the internal reset and retrieve the response.

Level 5: Remote Code Execution (RCE)
Vuln: Remote Code Execution

Level: 5

Cause: The GET /admin/stats/disk endpoint accepts a parameters value that is passed to a system command without sanitization. Only the Chef role is authorized to call it.

Process: Use the Chef token obtained from the SSRF step or the BOPLA privilege escalation. Send a request to the disk stats endpoint with a command injection payload in the parameters parameter.

Payload: GET /admin/stats/disk?parameters=;id with a Chef Bearer token. The response contains the output of the id command, confirming command execution. Replace id with any system command, such as a reverse shell or file read.

Broken Authentication: JWT Forgery
Vuln: Broken Authentication

Level: Not level based; affects the authentication layer

Cause: The API uses JWT with the HS256 algorithm, but the signature is not verified. The token payload contains the username in the sub claim, and the server trusts the decoded value without checking the signature.

Process: Decode your own JWT, change the sub claim from your username to a privileged username such as chef, and send the modified token with the signature section left empty. The API accepts the token and authenticates you as the chosen user.

Payload: A token with a modified payload such as {"sub": "chef"} and an empty signature after the trailing dot. Use the token against GET /profile to confirm the impersonation.

Unrestricted Resource Consumption
Vuln: Unrestricted Resource Consumption

Level: Not level based; affects the authentication and reset flow

Cause: The POST /reset-password endpoint has no rate limiting. An attacker can repeatedly trigger password reset PINs for a known username.

Process: Send many rapid requests to the reset endpoint with a valid username. The API sends a new PIN each time, causing SMS costs, inbox spam, and potential denial of service.

Payload: A loop of POST /reset-password requests with a body such as {"username": "chef"}. No throttling or CAPTCHA is present.

Broken Function Level Authorization (BFLA)
Vuln: Broken Function Level Authorization

Level: Not level based; affects menu and admin functions

Cause: Several endpoints check authentication but not the caller's role. A Customer token can reach functions intended for Chef or admin roles.

Process: Use a Customer token against endpoints such as PUT /menu, DELETE /menu/{id}, or GET /admin/stats/disk. Compare the response with a Chef token to confirm the missing role check.

Payload: PUT /menu or DELETE /menu/17 with a Customer Bearer token. A successful response confirms that function level authorization is not enforced
