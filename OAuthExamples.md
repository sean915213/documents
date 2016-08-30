# Summary
In this document I will provide some general guidance on how I envision OAuth2 working for 2 different providers.

## Google
Google's OpenId implementation (Google Sign In) is built on top of OAuth2.  The relevant documentation is here: https://developers.google.com/identity/.

### New Device Registration
The Google OAuth2 flow for Stocksnips would proceed in the following manner:

   1. To register use Google's SDK (or any other method) to obtain a `authorizationCode`.  This is a one-time code that the API can use to authenticate the user.
   2. Send the `authorizationCode` to the API after logging the user in (with a request that also indicates Google is the provider).
   3. The API will exchange the `authorizationCode` with Google to obtain an access token, refresh token, and expiration time.
   4. The API will use the access code provided to fetch the user's OpenId information.  If successful this response returns information like user name, email address, issuer subject, etc.
   5. The API will check whether this user is already registered by using the issuer subject.  Depending on whether the user is registered the following happens:
      - If the user is already registered a new device registration is created and linked using the new Google OAuth info.
      - If the user is not already registered a user entry and device registration are created.
      - **Note** - This exact flow may need modified depending on when the subscription is processed.
   6. The API generates and provides a new `accessToken` and `refreshToken` for the device.

### Token Refresh
All requests after registration will include the provided `accessToken` to authenticate.  If this token has expired then the API will return the appropriate response.  The app will then attempt to refresh the token.  If the user's auth provider is Google then the API would take the following steps:
   1. Use the `provider_refreshToken` to ask Google to refresh the `provider_accessToken`.
   2. If this fails it indicates the user has revoked permission for the app.  The API will return the appropriate unauthorized response per the RFC.
   3. If the refresh succeeds the API stores the new `provider_accessToken` and generates a new `accessToken` and expiration time.  The new `accessToken` is returned to the client.
   
## Facebook
The Facebook implementation (Facebook Sign In) works slightly different from Google.  The relevant documentation is here: https://developers.facebook.com/docs/facebook-login/.

### New Device Registration
The Facebook OAuth2 flow for Stocksnips would proceed in the following manner:

   1. To register use Facebook's SDK (or any other method) to obtain an `accessToken`.  This access token is the actual `provider_accessToken`. This flow is different from Google in that way.
   2. Send the access token to the API after logging the user in (with a request that also indicates Facebook is the provider).
   3. The API will use the access code provided to fetch the user's profile information.  If successful this response returns information like user name, email address, etc.  Note in Facebook's case there is no issuer subject so this field would be the returned email.
   4. The API will check whether this user is already registered by using the issuer subject.  Depending on whether the user is registered the following happens:
      - If the user is already registered a new device registration is created and linked using the new Facebook OAuth info.
      - If the user is not already registered a user entry and device registration are created.
      - **Note** - This exact flow may need modified depending on when the subscription is processed.
   5. The API generates and provides a new `accessToken` and `refreshToken` for the device.

### Token Refresh
All requests after registration will include the provided `accessToken` to authenticate.  If this token has expired then the API will return the appropriate response.  The app will then attempt to refresh the token.  If the user's auth provider is Facebook then the API would take the following steps:
   1. Use the `provider_accessToken` to query the user's profile.
   2. If this query fails the user's token could have expired or they may have revoked access.  To determine this the API would attempt to 'extend' the token lifespan (this is Facebook's idea of a token refresh).
   3. If the extension succeeds the API stores the new `provider_accessToken` and generates a new `accessToken` and expiration time.  The new `accessToken` is returned to the client.
