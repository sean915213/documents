# In App Subscriptions
This document explains the proposed flow for Stocksnips In-App Subscriptions.

## Proposed Classes
The following new classes will be used on the app to work with subscriptions.
#### Subscription
The `Subscription` class will contain the most basic information about a subscription level. It will be filled with information from the `OSG_SubscriptionTypes` table.  This class will have the following properties:
 - `description` - A textual description of this subscription level.
 - `name` - The name of this subscription level.
 - `portfolioLimit` - The number of portfolios allowed in a user's holdings for this subscription level.
 - `fee` - The monthly cost of this subscription.
 - `appleProductId` - The associated Apple product identifier for this subscription.
 - `googleProductId` - The associated Google product identifier for this subscription (need more info on this).
 
 #### UserSubscription
 The `UserSubscription` class will inherit from `Subscription`.  It will contain additional information specific to a user's current subscription.  In addition to those from `Subscription`, this class will have the following properties:
 - `vendor` - The 'vendor' that is providing this subscription.  Possible values would be `apple` or `google`.
 - `expiration` - The date when this subscription expires.
 
 #### SubscriptionOrder
 The `SubscriptionOrder` class will be used to submit new subscription data to the API after a subscription has been purchased.  The following properties will be included:
 - `vendor` - The 'vendor' that is providing this subscription.  Possible values would be `apple` or `google`.
 - `receiptData` - This will be the data provided by the vendor which will allow the API to validate the purchase and subsequently check for subscription lapses.
 
 ## API Changes
 The following changes will be made to the API in order to support subscriptions.
 
 #### Products Endpoint
 A new endpoint will be created at 'subscription/products'.  This endpoint will return a list of `Subscription` objects.  The apps will use this endpoint to display the available subscriptions when allowing the user to purchase a new subscription.
 
 #### Order Endpoint
 A new endpoint will be created at 'subscription/order'.  This endpoint will be used to submit new subscription data.  The following logic will be followed:
 - After a successful purchase the app will send an instance of `SubscriptionOrder` to the API.
 - The API will validate and setup the user's subscription using the logic in [Subscription Validation](#subscription-validation).
 - If validation fails the API will send an appropriate error message.  Otherwise, the API will return an instance of `UserSubscription` with the new subscription info.
 
 #### User Profile Fetch
 Currently the app fetches the user's profile (via user/login) in the form of a `LoginServiceResponse` after retrieving/obtaining an access token.  A property should be added to this response called `subscription`.  The API will provide this value using the following logic:
 - If subscription information exists for user the API should follow the steps in [Subscription Validation](#subscription-validation) to ensure the subscription has not expired or been cancelled.
 - If subscription information does not exist the `subscription` property should be returned blank.
 
 #### Token Refresh
 In addition to the current logic performed during a token refresh the API should also [validate the user's subscription](#subscription-validation).
 
 ## App Changes
 The following changes will be made to app logic.
 
 #### Subscription Check
 When fetching the user's profile the app will now need to check for the `subscription` object.  If this object does not exist the app should redirect the user to a flow that allows purchasing a subscription rather than to the user's list of portfolios.
 
 ##<a name='subscription-validation'></a>Subscription Validation
 The following is the logic that the API should follow in order to validate an in-app purchase subscription. **If these validation methods explicitly fail the subscription information should be removed from the db**. Note: explicit failure does not include incidental failures in the pipeline (e.g. unable to contact the vendor's servers for validation). 
 
 #### Apple Validation
 The 'receipt' provided by Apple after a successful in-app purchase (or via subsequent validations) is simply a base64 encoded string.  This string can be validated by following these steps:
 - A POST request is made to https://buy.itunes.apple.com/verifyReceipt (or https://sandbox.itunes.apple.com/verifyReceipt in development).  This request contains the following fields:
   + `receipt-data` - The base64 receipt provided by the app.
   + `password` - A password shared between the API and Apple.
 - The API should first validate that the resulting response has an http code of `200` and a `status` property of `0`.
 - The API should then pull out the property `latest_receipt_info`.  This is an array of JSON 'receipt' objects describing the subscription purchases.  The following logic should be performed:
   + Find the latest receipt by sorting the receipts on `expires_date_ms`.
   + Determine whether the latest `expires_date_ms` has lapsed.  If it has the subscription has expired and **not** been renewed.
 - If the above checks pass then the subscription is valid.  The API should now do the following to update the subscription values in the db:
   + Using the latest receipt found above pull out the property `product_id`.  This corresponds with the subscription's `appleProductId`.  Make sure this value matches the current user's subscription.  If it does not then update the user's subscription.
   + Save the `latest_receipt` value from Apple's response in the db under the user's `subscriptionReceiptData`.
   + Save the `expires_date_ms` value in the db under the user's `subscriptionExpiration`.
 
   
