[<-- BACK](https://github.com/bkieselEducational/OAuth-Concepts-and-Implementation)
# OAuth-2.0-from-Scratch<br>

### OAuth 2.0 Flow Diagram
<img width="1504" alt="oauth_flow" src="https://github.com/bkieselEducational/OAuth-2.0-from-Scratch/assets/131717897/668b5262-0ffa-439d-bf04-c615ac763fd5"><br>
<br>
### OAuth 2.0 Flow Walkthrough
Step 1: The user clicks the OAuth Login button that we have supplied in the client side code. For security reasons we do not want to generate the OAuth flow initializing URL on our frontend as this would expose sensitive credentials.<br>
<br>
Step 2: The backend endpoint will pull our Google OAuth client credentials from our .env file and will use these values to initialize a Client class to be used for our flow. Now we must generate a number of random values to use for authorization during the flow. While all of these values are technically optional, it is the recommendation by the OAuth community to use them everytime! These values are as follows:
- state: A random string value that will act as our CSRF protection for the flow.
- nonce: This value can be considered CSRF protection for our OpenID Token that we will request. it is sent back in the JWT claims object for verification.
- code_verfier: A random alphanumeric string of varying length. The constraints by the OAuth community specify 43-128 characters in length. We save this value to be used later.
- code_challenge: The result of taking the code_verifier string and SHA256 hashing it. Then taking that hash and base64URL encoding it. We will send this in the initial request.<br>
<br>
Step 3: With our random values in tow, we must now construct our URL for Google. It will need to contain some additional parameters. Some are required some are optional:<br>

- access_type: offline (OPTIONAL: Setting this parameter to 'offline' tells Google to send a Refresh Token)
- scope: https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.profile openid email(REQUIRED: AT least one is required. Here we have 3. We want profile info email address and an openid token)
- prompt: select_account consent (OPTIONAL: Note that if we do not set the 'select_account' value here, there is no garauntee that a user that is logged into 1 Google account will be prompted to choose another when logging in. This is a bad user experience and from my viewpoint, setting the 'prompt' property should not be optional. But it is!)
- response_type: code (REQUIRED)
- redirect_uri: /your/oauth/callback/endpoint/path (REQUIRED)
- client_id: your_google_oauth_client_id (REQUIRED and sensitive. Gaurd it!!)<br>
<br>
With these values, we can construct an appropriate URL to begin the OAuth flow. Our Redirect URL will look something like the following:
<img width="851" alt="oauth_first_url" src="https://github.com/bkieselEducational/OAuth-2.0-from-Scratch/assets/131717897/40f5e57c-9e9b-4319-838d-8ff6453a9726"><br>
<br>
Step 4:
