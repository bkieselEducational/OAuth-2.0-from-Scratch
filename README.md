[<-- BACK](https://github.com/bkieselEducational/OAuth-Concepts-and-Implementation)
# OAuth 2.0 from Scratch<br>
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

## The Cryptography of OAuth 2.0
So regardless of any details above or in other parts of this repo, how the PKCE portion of our flow is actually implemented may continue to be a mystery. By what means is our package generating these values for us? If we needed to generate them ourselves, how would we do that? Let's explore these topics below.

```python
  # From flow.py in the google_auth_oauthlib library

class Flow(object):
    ...
    ...
    
    def authorization_url(self, **kwargs):
        """Generates an authorization URL.

        This is the first step in the OAuth 2.0 Authorization Flow. The user's
        browser should be redirected to the returned URL.

        This method calls
        :meth:`requests_oauthlib.OAuth2Session.authorization_url`
        and specifies the client configuration's authorization URI (usually
        Google's authorization server) and specifies that "offline" access is
        desired. This is required in order to obtain a refresh token.

        Args:
            kwargs: Additional arguments passed through to
                :meth:`requests_oauthlib.OAuth2Session.authorization_url`

        Returns:
            Tuple[str, str]: The generated authorization URL and state. The
                user must visit the URL to complete the flow. The state is used
                when completing the flow to verify that the request originated
                from your application. If your application is using a different
                :class:`Flow` instance to obtain the token, you will need to
                specify the ``state`` when constructing the :class:`Flow`.
        """
        kwargs.setdefault("access_type", "offline")
        if self.autogenerate_code_verifier:
            chars = ascii_letters + digits + "-._~"
            rnd = SystemRandom()
            random_verifier = [rnd.choice(chars) for _ in range(0, 128)]
            self.code_verifier = "".join(random_verifier)

        if self.code_verifier:
            code_hash = hashlib.sha256()
            code_hash.update(str.encode(self.code_verifier))
            unencoded_challenge = code_hash.digest()
            b64_challenge = urlsafe_b64encode(unencoded_challenge)
            code_challenge = b64_challenge.decode().split("=")[0]
            kwargs.setdefault("code_challenge", code_challenge)
            kwargs.setdefault("code_challenge_method", "S256")
        url, state = self.oauth2session.authorization_url(
            self.client_config["auth_uri"], **kwargs
        )

        return url, state
```

### code_verifier
In the code above we can see how this value is being generated. Note that the length constraint is typically recommended to be 43-128 characters.

1. First of all, we have defined an alphabet for the construction of the random string as:<br>
"abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789-._~"
2. Secondly, we instantiate an instance of SystemRandom() which we will use to select a characteer at random from the alphabet.
3. Using a list comprehension, we then generate a list of random characters, defaulting to the maximum length of 128 characters.
4. Finally, we join the list into a string of the random characters.

An example code_verifier: rstpcP5VkyNmRkPHlIhqz2-p2JghkQgwCRrXniQl-udv.MybF2Pmr2HRkoXfT7W0aywQracDnxHY~2_lCuDsljZWoCF9~Wur6uIyXkxENxafTturzNSUfbXH9K~FNFC8

### code_challenge
Now it is time to hash our random string.

1. First we instantiate a new hash object.
2. Next we update the hash object with the content of our code_verifier. As is typical of hash functions, we want the data to be input as raw bytes (python defaults to UTF-8), so to this end we are passing the code_verifier as str.encode(self.code_verifier)
3. Next we call the digest() method to run the hash function and return a list of raw bytes of no particular codec. Note that the sha256 algorithm will always return a list of 32-bytes or 256 bits!
4. Now we encode these random bytes as base64URLsafe() bytes. As base64 translates 8-bit bytes into 6-bit bytes, it will result in a longer string than we would have if wew used UTF-8 or some otheer codec. In our case, we will get 43 bytes of output. (If applicable, that output will be shortened from the end if there are any padding characters, when we call the code on the following line.)
5. Time to decode the bytes into a glyph (string) representation and strip off the trailing padding characters '=' (there will be no nore than 2 of these) as Google forbids them and they are not required to compute the hash on the server side.

An example of a code_challenge: xftkieDXjg_paZetCSFmsgRf1McELR5Ney-EaMTvrJ8

## Additional Resources
1. [OAuth 2.0 Official](https://oauth.net/2/)
2. [OAuth 2.0 Parameters](https://github.com/bkieselEducational/OAuth-2.0-Parameters)
3. [Google Oauth 2.0 API Reference](https://github.com/bkieselEducational/OAuth-2.0-API---Google-Reference)
