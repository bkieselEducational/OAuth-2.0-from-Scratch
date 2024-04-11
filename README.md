[<-- BACK](https://github.com/bkieselEducational/OAuth-Concepts-and-Implementation)
# OAuth-2.0-from-Scratch<br>
<br>

## OAuth 2.0 Flow Diagram
<img width="1504" alt="oauth_flow" src="https://github.com/bkieselEducational/OAuth-2.0-from-Scratch/assets/131717897/668b5262-0ffa-439d-bf04-c615ac763fd5"><br>
<br>

## OAuth 2.0 Flow Walkthrough

### Step 1:
**The user clicks the OAuth Login button that we have supplied in the client side code. For security reasons we do not want to generate the OAuth flow initializing URL on our frontend as this would expose sensitive credentials.**<br>
<br>

### Step 2:
**The backend endpoint will pull our Google OAuth client credentials from our .env file and will use these values to initialize a Client class to be used for our flow. Now we must generate a number of random values to use for authorization during the flow. While all of these values are technically optional, it is the recommendation by the OAuth community to use them everytime! The use of the code_verifier and code_challenge falls under the security concept defined by the OAuth consortium as PKCE: Proof Key for Code Exchange. Originally developed for use with Mobile Apps, it is now recommended in all flows to protect against Authorization Code Injection Attacks. These values are as follows:**
- state: A random string value that will act as our CSRF protection for the flow.
- nonce: This value can be considered CSRF protection for our OpenID Token that we will request. It is sent back in the JWT claims object for verification.
- code_verfier: A random alphanumeric string of varying length. The constraints by the OAuth community specify 43-128 characters in length. We save this value to be used later.
- code_challenge: The result of taking the code_verifier string and SHA256 hashing it. Then taking that hash and base64URL encoding it. We will send this in the initial request.<br>
<br>

### Step 3:
**With our random values in tow, we must now construct our URL for Google. It will need to contain some additional parameters. Some are required some are optional:**<br>

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

### Step 4:
**Using the URL generated above, we now redirect the client's browser to the Google 'Select Account' Login Screen.**<br>
<br>

### Step 5:
**Upon successful Login by our user, Google now returns a very short-lived code to our Redirect URI which we have registered with Google. It should also include our state parameter value that we sent earlier. We will match this value to what we saved in order to confirm that the response is coming from Google. Then we will respond to this request by immediately sending a JSON encoded POST request to another Google OAuth endpoint. The 'https: //oauth2.googleapis.com/token' endpoint!! That request should contain the following parameters:**<br>

- client_id: your_google_oauth_client_id (REQUIRED and Same as Above)
- client_secret: Our Google client secret given to us during the registration on GCP.
- redirect_uri: Because all OAuth flow related requests have this! (REQUIRED)
- code_verifier: The value that we sent the hash of in the original request. Now Google can use this to validate that code_challenge that we sent! (OPTIONAL but recommended!)
- grant_type: authorization (REQUIRED)
- code: The ephemaral code that Google just sent us! (REQUIRED)<br>
<br>
And the request should look something like:<br>
<br>
<img width="829" alt="oauth_code_exchange" src="https://github.com/bkieselEducational/OAuth-2.0-from-Scratch/assets/131717897/a3f7a031-74fa-4c49-a7c5-fde395e63335"><br>
<br>

### Step 6:
**Upon a successful reply by Google, we should now have 2 very important pieces of data. 1) An Access Token, which we can use as our Session Token if we choose or even use it to access Google's API on behalf of our Logged in user. Other options include discarding it as the user has been authenticated and now we can use our own Session Management to handle the Login. 2) We should also have received an ID Token in the form of a JWT. I always recommend verifying the signature. Above that, you can see if the Claims object has returned the same 'nonce' value that you sent earlier. If it does not match, you should abort this Login!**<br>
<br>

### Step 7:
**Finally, once you have managed the Login/Signup logic for the user, it is time to redirect their browser one last time to return them to your application as a Logged in user!**<br>
<br>

## Additional Resources
1. [OAuth 2.0 Official](https://oauth.net/2/)
2. [Google Oauth 2.0 API Reference](https://github.com/bkieselEducational/OAuth-2.0-API---Google-Reference)
