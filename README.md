# Citrix Workspace Microapps Middleware Sample for O365

This repository contains a fully functional sample that shows shows how
Microsoft Office 365 can be integrated into Citrix Workspace using
middleware. The sample is implemented using Flask.

The implemented features are:
- A microapp integration bundle, ready to import and use in your Microapp environment.
- Calendar functionality:
    - Calendar and event view over the next seven days.
    - Notifications for upcoming meetings.
    - Join virtual meetings directly from within Workspace.
    - Notifications for new meeting requests.
    - Respond to meetings Accept, Tentative, Decline with optional text.
    - Delete meeting events without responding.
    - Direct access to events in web-based Outlook.
- Email functionality:
    - Notifications about unread high-priority emails.
    - Notifications about unread emails from a Workspace user's manager.
    - Direct access to email in web-based Outlook.
    - Send Emails.
- Near real-time experience powered by Microsoft Graph event subscription
  and Microapps webhook listeners.
- Multi-tenant middleware service, built using Flask, that scales to being used by
  many Workspace and O365 environments.

Screenshots of the Microapps:
| <b>Standalone Actions</b><br> <img width="50%" alt="actions" src="https://user-images.githubusercontent.com/4073077/93028702-6e137c80-f60d-11ea-8e22-48f5628532d0.png"> | Send Email <br> <img width="50%" alt="send_email" src="https://user-images.githubusercontent.com/4073077/93028705-710e6d00-f60d-11ea-8657-96e22dd8c02c.png"> |
|:-:|:-:|
| <b>View Calendar</b> <br> <img width="50%" alt="view_calendar" src="https://user-images.githubusercontent.com/4073077/93028981-a87e1900-f60f-11ea-93f1-ed38b48005e6.png"> | <b>View calendar event</b> <br> <img width="50%" alt="view_calendar_event" src="https://user-images.githubusercontent.com/4073077/93029080-7ae59f80-f610-11ea-895c-e99e4dd88267.png"> |

## Getting Started

### Prerequisites
- Citrix Workspace with Microapp entitlement - [refer to this getting started guide](https://developer.cloud.com/citrix-workspace/build-workspace-microapp-integrations/docs/getting-started) for setting up a free developer instance
- Office 365 - note that the [Microsoft 365 Developer Program](https://developer.microsoft.com/en-us/microsoft-365/dev-program) comes with a free developer test instance
- Admin access to AAD (comes with Office 365)
- A publicly routed host to run the Flask middleware (any public cloud will work nicely, see [Flask Deployment Options](https://flask.palletsprojects.com/en/1.1.x/deploying/). Or use a free Azure Tier as described below.)

### Get and deploy the multi-tenant service


The following steps describe how the middleware can be deployed on top of Azure as a webapp in the free tier for testing
If you don’t have an Azure subscription, you can create a [free account](https://azure.microsoft.com/en-gb/free/).
Install the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).

```
# Login to Azure
az login

# Clone the repo
git clone https://github.com/robertbreker/microapps-middleware-sample-o365.git
cd microapps-middleware-sample-o365.git

# Choose an <app_name> that's unique across Azure, e.g. my-microapps-middleware-sample-o365
# In application.py, update SUBSCRIPTION_CALLBACK_URL to match your service
# SUBSCRIPTION_CALLBACK_URL = 'https://<app_name>.azurewebsites.net/'

# Login to Azure
az login

# Deploy the app on a Free Tier (Replace <app_name> with a unique name across Azure)
az webapp up --sku F1 -n <app_name>
# Your app will now be running and be ready to be used at https://<app_name>.azurewebsites.net/
# Note that the command also reveals the resourcegroup, that you will need in subsequent commands.

# Configure the startup command to include multiple gunicorn workers.
# Multiple workers are needed so that one worker process can validate MS Graph webhook subscriptions while another creates them.
az webapp config set --resource-group <your-resource-group> --name <app_name> --startup-file "gunicorn --bind=0.0.0.0 --timeout 600 application:app --workers=4"

# If you don't need the webapp anymore, you can delete the resource group later as follows.
# az group delete --name <your-resource-group>

```
Note that this middleware service, once deployed, can be used by multiple Citrix Workspace tenants and configured integrations.
All that's needed is the middleware's URL.

### Deploy the integration to your Citrix Workspace Microapp environment
For each tenant (combination of O365 and Citrix Workspace deployment) the following steps need to be followed.

Configure an app client and secret for Microsoft Graph using the Azure Portal.

1. Open [App registrations](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredApps]) in your Azure Active Directory
2. Create a "New registration"
3. Enter a user facing name, such as "microapps-middleware-sample-o365"
4. Complete the operation by selecting "Register"
5. Take note of the "Application (client) ID", that is the client id needed for Workspace later
6. Select "Certificates & secrets" from the left menu
7. Select "New client secret"
8. Provide a descriptive name, such as "workspace-integration" and click "Add"
9. Take note of the "Value", i.e. the secret, that is the client secret needed later to configure Citrix Workspace

Configure the API permissions for the new app client. 

1. Select API permissions in the left menu.
2. Select Add a Permission -> Microsoft Graph -> Application Permission
    1. Tick Calendars read
    2. Tick Mail.read
    3. Tick User.Read.All
3. Select the following delegated permissions
    1. Tick Calendars.ReadWrite
    2. Tick Mail.Send
4. Select Add permissions to complete the operations
5. Select "Grant admin consent" to activate the permissions
 
 Configure the app client authentication.
1. Select "Authentication" from the left menu
2. Select "Add a platform", "Web" and add two redirect URIs
    1. https://\<yourcustomerid\>.\<yourgeo\>.iws.cloud.com/app/api/auth/serviceAction/callback
    2. https://\<yourcustomerid\>.\<yourgeo\>.iws.cloud.com/admin/api/gwsc/auth/serverContext

Import and configure the integration bundle.

1. Login to [Citrix Cloud](https://cloud.citrix.com/)
2. Select "Manage" for the Microapps tile
3. Select "Manage", "Add Integration", "Import a previously configured integration"
4. Select SampleO365Middleware.service.mapp and "Import"
5. Right click the Integration, select "Edit", in the left menu "Configuration"
6. In the "Base URL", replace 'app-name' with your azure webapp name
7. In the "Service Authentication" section, find "Token URL" and modify the path, replacing the string 'your\_tenant\_id' with your "Tenant ID" for your Azure Active Directory
8. In the "Service Action Authentication" section find the "Token URL" and "Authorization URL" and modify the path, replacing 'your\_tenant\_id' with your "Tenant ID" for your Azure Active Directory
9. Configure the previously noted "Client ID", and "Client Secret"
10. Also configure the "Client ID" and "Client Secret" for Service Action Authentication
11. Apply the settings by click "Save" in the bottom left
12. Select "Webhook Listeners" and copy the URLs of email\_webhook and calendar\_webhook, remembering which is which
13. Select "Data loading" and edit the "trigger\_middleware" data endpoint
14. Edit the query parameters to contain the calendar\_webhook and email\_webhook URLs, that you copied previously
15. Subscribe users to the integration and test the integration

## Design notes

The middleware \([application.py](application.py)\), sits between Microapps and
Microsoft Graph. It registers for event subscriptions from Graph, processes the
callbacks, and forwards processed data to Microapps. The integration bundle
\([SampleO365Middleware.service.mapp](SampleO365Middleware.service.mapp)\) implements a Microapp integration that
receives this processed data and presents it to Workspace users.

The main implementation details are as follows:

  - Microapps implements the SOR authentication without involving the middleware
  - The middleware is regularly called by a data-endpoint in Microapps, which now has a
    valid bearer token for the SoR
  - The middleware caches the bearer token
  - The middleware subscribes to SoR events and renews subscriptions every 24h
  - The subscription is done with a separate API call for each user
  - When the SoR calls back with events, the middleware makes subsequent calls
    to the SoR for obtaining the actual data of interest
  - Service actions are passed through to the SoR by the middleware
