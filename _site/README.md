# PagerDuty Bulk User Management Tool

This is a standalone tool, suitable for use as a PD add-on to import, export and edit users. The code for this project showcases [PKCE OAuth](https://v2.developer.pagerduty.com/docs/oauth-2-functionality-pkce) as well as the [users](https://api-reference.pagerduty.com/#!/Users/get_users) endpoint of the PagerDuty API.

## How to use
A live version of the tool is running [here](https://pagerduty-samples.github.io/pagerduty-bulk-user-mgr-sample/).
```
https://pagerduty-samples.github.io/pagerduty-bulk-user-mgr-sample/
```
## Explanation of the Code

### PKCE OAuth Sample
This project is also a great example of using [PKCE OAuth](https://oauth.net/2/pkce/) that is available with authorizing access to PagerDuty account data. This section will explain the OAuth code. In traditional grant flow OAuth there is a client id and secret that are used to verify the identity of the application requesting access.

The issue with this flow is that, as a developer, you need to conceal the client secret to keep it safe. That usually meant keeping the secret on the server, rather than in the client. If the rest of the application can run in the client, creating extra infrastructure to simply hold the secret may seem like overkill. Now, with PKCE, you no longer need to store a secret. Instead, you deal with a dynamic challenge that is different each time that the auth flow runs. This is how it works.

#### Create PKCE Verifier and Challenge
First, you generate code verifier. The PKCE standard calls for a random 128byte, base64 urlEncoded value. For this project, we initialize a new TypedArray with `Uint8Array(128)`, then use `window.crypto.getRandomValues(array)` to populate that array. 

```javascript
function gen128x8bitNonce() {
    const array = new Uint8Array(128); //( generate 1024bits 8*128
    window.crypto.getRandomValues(array);
    return array;
};

// generate code verifier
const generatedCode = gen128x8bitNonce();
```


The standard calls for the `generatedCode` to be `base64Unicode` encoded. So, this is done before saving it to the browser's `sessionStorage` as `code_verifier`.

```javascript
// base64 encode code_verifier
const codeVerifier = base64Unicode(generatedCode.buffer);        
// save code_verifier
sessionStorage.setItem('code_verifier', codeVerifier);
```

Using the value for `code_verifier` the next step is to create a `code_challenge`. This is created by hashing `code_verifier` with the following code, and then  passing that value to `base64Unicode` to encode it.

```javascript
async function digestVerifier(vString) {
    const encoder = new TextEncoder();
    const verifier = encoder.encode(vString);
    const hash = await crypto.subtle.digest('SHA-256', verifier);
    return hash;
}
// generate the challenge from the code verifier
const challengeBuffer =  await digestVerifier(codeVerifier);

// base64 encode the challenge
const challenge = base64Unicode(challengeBuffer); 
```

#### Request Auth Code
Using the `code_challenge` generated above, and the `client_id` and `redirect_uri` from your app's [OAuth Registration](https://v2.developer.pagerduty.com/docs/oauth-2-functionality) the `authUrl` is created below. A few key parameters to point out are the `response_type=code`, which tells PagerDuty to return an Authorization Code, and the `code_challenge_method=S256`. This tells PagerDuty by which method the challenge is hashed. In this case, SHA-256.

```javascript
const authUrl = `https://app.pagerduty.com/oauth/authorize?` +
                    `client_id=${APP_CONFIG.clientId}&` +
                    `redirect_uri=${APP_CONFIG.redirectUrl}&` + 
                    `response_type=code&` +
                    `code_challenge=${encodeURI(challenge)}&` + 
                    `code_challenge_method=S256`;
```



#### Request Auth Token
When PagerDuty returns the Authorization Code, it's time to request an Access Token. This is done by making a POST request to `https://app.pagerduty.com/oauth/token` as seen below. Notice, that we're now passing the original value of `code_verifier` that we generated earlier in the flow. This is done at the last step so that the server can gaurantee that otherwise state-less request is coming from the same client source--ensuring that the `authorization_code` sent from PagerDuty was not intercepted in the wild. The server will have stored the value for `code_challenge_method` from earlier and will use it along with `code_verifier` to validate the request.

```javascript
let requestTokenUrl = `https://app.pagerduty.com/oauth/token?` + 
                    `grant_type=authorization_code&` + 
                    `code=${urlParams.get('code')}&` +
                    `redirect_uri=${APP_CONFIG.redirectUrl}&` + 
                    `client_id=${APP_CONFIG.clientId}&` + 
                    `code_verifier=${sessionStorage.getItem('code_verifier')}`;
```
The response to the token request will return an Access Token object  with the following format.

```json        
{
    "access_token":"<access_token>",
    "token_type":"bearer",
    "refresh_token":"<refresh_token>",
    "scope":"user"
}
```
Save this object to `localStorage`. This sample did so with the following code:

```javascript 
localStorage.setItem("pd-token", (JSON.stringify(data)));
```

When making calls to the PagerDuty API using this Access Token set the  `Authorization Header` like so `Authorization: Bearer <access_token>`. Also,the `Accept:application/vnd.pagerduty+json;version=2` header is required when using a Bearer token. This ensures that you're using version 2 of the API.

# docker run --rm --volume="$PWD:/srv/jekyll" -p 4001:4001 jekyll/jekyll:pages jekyll serve 
# ssh -R 80:localhost:4000 ssh.localhost.run
