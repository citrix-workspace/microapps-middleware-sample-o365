
# Middleware Design Patterns
If a microapp integration requires a specific customisation, middleware is a great way to augment the capabilities of the Microapp Platform. Middleware sits between the Microapp Platform and the System of Record (SoR) and can perform a service:

- Translating/transforming http requests from the Microapp Platform to the SoR or vice versa.
- Webhook data injection and lifecycle management.

The middleware itself can take different forms, serverless functions, web applications, online database services etc. This article outlines some of the possible design patterns you can use to expand the range of use-cases for microapp integrations.

## Translating/Transforming Http Requests

![Transformation](/Transformation.png)

This design pattern uses serverless functions in the middleware to provide an intermediary service between the Microapp Platform and the SoR. The integration on the Microapp Platform must set the base url to point to the middleware service where functions can be exposed at different URIs and triggered via http requests. The middleware functions can be triggered using Data Loading endpoints or Service Actions from within the Microapp Platform builder. What kind of services can the serverless functions provide?

- transforming parameter data-types in the http request
- translating between the Microapp Platform (which uses a REST API) and SoRs which use non-RESTful APIs, such as SOAP.
- exposing web resources that have no API at all!

The video below demonstrates how serverless functions can be used to obtain data from a web resource that is not accessible via an API. In the video, a data loading endpoint is configured to trigger a middleware function that offers an html websraping service. In this instance, an html canteen menu is obtained and parsed to find the daily specials. The daily specials can then be neatly transformed into a json object and delivered to the Microapp Platform in the body of an http request. In this way, the middleware implements a REST API which the web resource does not offer.

<iframe id="ytplayer" type="text/html" width="320" height="180" allowfullscreen="1"
  src="https://www.youtube.com/embed/yVvuymHhGpk?autoplay=0&modestbranding=1&origin=https://developer.cloud.com/"
  frameborder="0"></iframe>
</body></html>

## Webhook Data Insertion
A useful feature of integrations on the Microapp Platform is the ability to create webhook listeners, http endpoints that can receive incoming requests and insert json data into cache tables. In combination with middleware, this feature enables use cases which rely on SoR webhooks APIs. For demonstration purposes we have built a microapp integration and multi-tenant middleware which you can download from our [public GitHub repository](https://github.com/citrix-workspace/microapps-middleware-sample-o365) using Microsoft Outlook as the SoR.

![Data Insertion Architecture](/WebhookDataInsertion.png)

Webhook data insertion middleware must implement:

  **Managing the webhook API lifecycle**  
  The middleware offers an http-triggered service that creates subscriptions to events on the SoR using their webhook API. This service can be triggered from the Microapp Platform. Note that the http request must contain at least one webhook listener url, needed by the middleware in order to know where to return information in the future. Once webhook subscriptions are created, there is typically an upper limit on their lifetime - the middleware must also keep track of expired subscriptions and handle their renewal.

  **Receiving webhook callbacks**  
  The notification url (the destination address for webhook callbacks) must also be hosted in the middleware. This is because follow-up calls to the SoR are necessary to gain the full context of an event. For example, a subscriptionis created to 'sent email' events on the SoR. A callback is received as a result of a user sending an email but certain suplementary information is required, "what is the subject?" or "is this email of high importance?". The middleware makes these clarifying calls to the SoR before bundling together all the information required by the microapp user in a json object which can be sent to a webhook listener on the Microapp Platform.

Webhook data insertion middleware may implement:

  **Multi-tenancy**  
  If you require your middleware to be shared across multiple 'customers' on the Microapp Platform, it must implement a mechanism to store the state of the different customers is serves. This allows it to answer the question "should I be notifying customer A or B about this event?". The state can be stored using a database or other persistent storage mechanism.

  **Service Actions**  
  Service actions are initiated by the end user within Citrix Workspace and trigger an http request from Citrix Workspace to the Microapp Platform. The Microapp Platform then sends the request to the middleware which implements a simple passthrough service. The passthrough service receives http requests and forwards them to the SoR without modification (unless required) of the message body or headers.

# Middleware and Authentication
There are many design choices regarding authenticaion and middleware:

-  Intercepting the authentication process between the Microapp Platform and the SoR.
-  Complete separation from the Microapp Platform and SoR authentication process.
-  Implementing its own authentication directly with the SoR without the involvement of the Microapps platform.

The most simple option to implement is complete separation of the middleware from the Microapp Platform and Sor authentication process. Taking the following example of an OAuth 2.0 Client Credentials flow:

![Authentication Details](/authentication.png)

The client credentials dance is performed in the backend of the Microapp Platform and ensures that all http requests emitted will have an up-to-date bearer token in their authentication header. The challenge that arises is in how to ensure that the middleware is also always in possesion of an up-do-date bearer token? A simple solution is to regularly poll the middleware at intervals of less than the token lifetime. In this way, the middleware receives a regular cadence of http requests with up-to-date bearer tokens in the security header. The middleware can then store these tokens in a database. 
