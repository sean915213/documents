# Summary
The goal of this document is to comprehensively describe the new authentication flow for the Stocksnips app.  The goal of this new flow is to combine the current basic-token hybrid scheme with the authentication provided by OAuth2 providers like Google and Facebook.

## Overall Registration & Authentication Flow
Below will be the new general process for registering a new user and keeping them authenticated on a device.  The required changes to the app and the database to implement this flow will be detailed below.

### Registration Flow
#### Step 1
Upon opening the app on a device for the first time the app will recognize that it has no saved credentials.  In this case it should present the user with the option to login to an existing account or create a new one. If the user chooses to create a new account then we must gather the relevant account information.  This is done via one of two methods:
   - **Create Stocksnips Account** - This flow will be very similar to our current registration system.  The user fills out the relevant profile information like full name, email, etc.
   - **Login via Facebook, Google, etc** - This flow will utilize the relevant OAuth2 provider's SDK to authorize the user for the app.  The only required result will be a token that the app shares with the API.  The API will then be responsible for gathering the relevant profile information from the provider.
   
#### Step 2
The second step of registration will be choosing the registration plan.  For now this is the free trial.
   
**Possible Modification**: To simplify the app logic it may make sense to break the initial authentication and subscription selection into two different steps.  This would make most sense if the app is to detect expired subscriptions and take them through the selection process again at that point.

#### Step 3
The app will then submit the gathered information.  The submitted payload will indicate whether the user used manual sign up or used an OAuth2 provider.  If an OAuth2 provider was utilized the payload will contain the relevant token and identifier for the provider.

### Authentication Flow
All authentication done for the services post registration will utilize the OAuth2 Bearer scheme (https://tools.ietf.org/html/rfc6750#page-7). 

#### Authentication Tokens
Whether the user signed up manually or used an OAuth2 provider the end result will be the server providing two tokens that should be saved securely by the app:
  - `accessToken` - This token is similar to our current auth token.  It will be included in the authorization header for all requests requiring authorization.  This token will expire after some time period.
  - `refreshToken` - This token will be used when the access token expires to request a new token.  This token never expires and the only way to get a new token is to log in again.

#### Token Refresh
The app will use the `accessToken` to authorize all requests until it receives an authentication challenge.  This challenge will follow the format provided in the above RFC.  The app should respond to this challenge in the following manner:

   1. Immediately attempt refreshing the `accessToken` via the `refreshToken` by making the appropriate request to the API.
   2. The API will attempt refreshing the token depending on how the user registered:
      - If the user registered manually then the token is refreshed immediately and returned.
      - If the user registered via an OAuth2 provider then the API must make a call to this provider in order to determine whether their authentication is still valid.  This will generally happen by attempting to refresh that provider's respective access token (but not always- the providers differ).  If this check is successful then another `accessToken` is generated and returned.
   3. The app then proceeds based on the response:
      - If the API returns 200 OK with a new `accessToken` the app should immediately begin the request again with the new token.
      - Otherwise the app should interpret the response. Some responses should warrant a request to retry (e.g. no network) while others mean the user is explicitly no longer authorized (e.g. not authorized) and should be directed to login again.

## Database Changes
In order to support the above changes important modifications will need to be made to some database tables:

### User Table
The user table will have the following columns added:
   - `authProvider` - A string indicating which provider was used to authenticate the user.
   - `authSubject` - A string that uniquely identifies the user to the OAuth2 provider (if using OAuth2).  In most cases this is the user's email.  But for certain providers (e.g. Google) they make clear that the email address could change and the subject field should be used as the unique id.

### Device Registrations Table
Each of the user's devices is provided a unique `accessToken` and `refreshToken`.  These tokens are also (optionally) linked to OAuth2 tokens from a provider.  Because of these (and other reasons) each user will have multiple entries in this table.  The relevant columns will be:
   - `accessToken` - The token used to authenticate against the service.
   - `refreshToken` - The token used to refresh `accessToken`.
   - `tokenExpiration` - The time when the current `accessToken` will expire.
   - `provider_accessToken` - The (optional) access token used to authenticate against the OAuth2 provider.
   - `provider_refreshToken` - The (optional) refresh token used to refresh the `provider_accessToken`.
