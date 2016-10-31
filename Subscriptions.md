# In App Subscriptions
This document explains the proposed flow for Stocksnips In-App Subscriptions.

### Proposed Classes
The following new classes will be used on the app to work with subscriptions.
#### Subscription
The `Subscription` class will contain the most basic information about a subscription level. It will be filled with information from the `OSG_SubscriptionTypes` table.  This class will have the following properties:
 - `id` - The description's id in our database.
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
 - `id` - The 'id' of the purchased subscription.
 - `vendor` - The 'vendor' that is providing this subscription.  Possible values would be `apple` or `google`.
 - `receiptData` - This will be the data provided by the vendor which will allow the API to validate the purchase and subsequently check for subscription lapses.
 
 ### API Changes
 The following changes will be made to the API in order to support subscriptions.
 
 #### Products Endpoint
 A new endpoint will be created at 'subscription/products'.  This endpoint will return a list of `Subscription` objects.  The apps will use this endpoint to display the available subscriptions when allowing the user to purchase a new subscription.
 
 #### Purchased Endpoint
 A new endpoint will be created at 'subscription/order'.  This endpoint will be used to submit new subscription data.
 
 #### User Profile Fetch
 Currently the app fetches the user's profile (via user/login) in the form of a `LoginServiceResponse` after retrieving/obtaining an access token.  A property should be added to this response called `subscription` that will contain an instance of `UserSubscription` if the user has a current subscription.
