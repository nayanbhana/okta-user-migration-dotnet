# okta-user-migration-dotnet

**Purpose**: *This project showcases how you can perform a just-in-time (JIT)
migration from your existing LDAP userbase into Okta) using an IIS server and
some .NET code.*


# User Migrations Made Simple

Migrating users to a new database can be thought of in two components: profile migration and credential migration.

- Profile migration is the migration of users and their metadata. This typically can be accomplished via an import and an API.
- Credential migration is the process of migrating a user's existing password, which can provide some difficulty.

When planning a migration, take into consideration how both profile and credentials will be
migrated. This configuration is designed to authenticate a user against an existing LDAP directory, upon successful authentication the user's credentials are then migrated to Okta and the user is signed into an OIDC application. The full project linked to this article can be used as a base for various migration methods thus can be used to migrate credentials from various sources.

For more information on the migration methods, please visit our [whitepaper](https://www.okta.com/resources/whitepaper-okta-user-migration-guide/) on user migrations.This document will cover a Hybrid Live migration. In this type of migration the following assumptions are made.

1. Users are stored in an existing database (LDAP)
2. Profile data will be imported and users will be in a "PROVISIONED" state
3. A custom login page on a hosted server will be used

## Prerequisites

To begin you will need the following:

* A Windows Server (2012 and up) with the Internet Information Systems (IIS) role [installed](https://docs.microsoft.com/en-us/iis/web-hosting/web-server-for-shared-hosting/installing-the-web-server-role)
* [ASP .Net 4.6](https://docs.microsoft.com/en-us/dynamics-nav/how-to--install-and-configure-internet-information-services-for-microsoft-dynamics-nav-web-client) and [.Net 4.71 Framework](https://dotnet.microsoft.com/download/thank-you/net471-developer-pack) installed
* Microsoft [Visual Studio Community 2017](https://visualstudio.microsoft.com/downloads/?utm_medium=microsoft&utm_source=docs.microsoft.com&utm_campaign=button+cta&utm_content=download+vs2017)(optional)
* Source of truth (LDAP)
* [Administrator](https://www.okta.com/resources/administration-exam-study-guide/#administrator-exam-subject-areas-10) level understanding of Okta
* Okta tenant, start [here](https://developer.okta.com/signup-now/) for free

## High Level Overview

![Okta Migration Flow](https://github.com/oktadeveloper/okta-user-migration-dotnet/raw/master/assets/okta-migration-flow.png)

At a high level, a prestaged user in a PROVISIONED state will browse to a custom login page that utilizes the Okta Sign In widget. When the user logins to the page, their credentials will be sent to the existing LDAP server. The credentials will be authenticated against the LDAP server. Upon a successful login & response, the Okta Users API will set the LDAP password as the User's Okta password to complete the credential migration process. A boolean attribute is used to determine a user's migration status.

Lets begin.

## 1.Okta Configuration

Create your free Okta developer tenant. Sign in as an admin for the following

1. Create an [API token](https://developer.okta.com/docs/api/getting_started/getting_a_token) (save for later)

   ![Okta Create Token](https://github.com/oktadeveloper/okta-user-migration-dotnet/raw/master/assets/okta-create-token.png)

2. Enable [CORS](https://developer.okta.com/docs/api/getting_started/enabling_cors) for your localhost app Example: http://localhost:10100

   ![Okta Add Origin](https://github.com/oktadeveloper/okta-user-migration-dotnet/raw/master/assets/okta-add-origin.png)

3. Using the [Profile Editor](https://help.okta.com/en/prod/Content/Topics/Directory/eu-profile-editor.htm?cshid=ext_Directory_Profile_Editor) create two new attributes:

   | Type    | Name             |
   |---------|------------------|
   | Boolean | isPasswordInOkta |
   | String  | migration_app    |

   ![Okta Profile Editor](https://github.com/oktadeveloper/okta-user-migration-dotnet/raw/master/assets/okta-profile-editor.png)

4. Create an [OIDC web app](https://help.okta.com/en/prod/Content/Topics/Apps/Apps_App_Integration_Wizard.htm?Highlight=app%20integration) with the following options, then copy the Client ID and Client Secret(save for later).

   1.Enable implicit flow
   2. Set "Login Redirect URI's" to `http://localhost:10100/authcallback`

      ![Okta Migration Demo App](https://github.com/oktadeveloper/okta-user-migration-dotnet/raw/master/assets/okta-migration-demo-app.png)

5. Create a [Group](https://help.okta.com/en/prod/Content/Topics/Directory/Directory_Groups.htm?cshid=Directory_Groups#Directory_Groups) called "Migration App"

   ![Okta Add Group](https://github.com/oktadeveloper/okta-user-migration-dotnet/raw/master/assets/okta-add-group.png)

6. Assign the"Migration App" group to the app created in step 4

   ![Okta Migration App Group Assignment](https://github.com/oktadeveloper/okta-user-migration-dotnet/raw/master/assets/okta-migration-app-group-assignment.png)
7. Create a [group rule](https://help.okta.com/en/prod/Content/Topics/Directory/Directory_Groups.htm) that states "IF user.migration_app equals "true" THEN Assign to "Migration App"

   ![Okta Migration App Edit Rule](https://github.com/oktadeveloper/okta-user-migration-dotnet/raw/master/assets/okta-migration-app-edit-rule.png)

8. Using the Okta API, create your users in a [PROVISIONED state](https://developer.okta.com/docs/api/resources/users#create-user-without-credentials)

   ![Okta Non-Migrated User](https://github.com/oktadeveloper/okta-user-migration-dotnet/raw/master/assets/okta-non-migrated-user.png)

   To Create provisioned user:

   ```
   POST /api/v1/users?activate=true HTTP/1.1
   Host: ORG.okta.com
   Accept: application/json
   Content-Type: application/json
   Authorization: SSWS TOKEN
   cache-control: no-cache
   Postman-Token: 18c2532b-2357-4adb-9755-24676877
   {
     "profile": {
       "firstName": "FIRST",
       "lastName": "LAST",
       "email": "FIRST.LAST@DOMAIN.com",
       "login": "FIRST.LAST@DOMAIN.com",
       "migration_app": "true"
       "isPasswordInOkta":false
     }
   }
   ```
   Response for provisioned user

   ```
   {
     "id": "00u9hwxxxxxxxxxxxxxx",
     "status": "PROVISIONED",
     "created": "2019-01-29T19:24:35.000Z",
     "activated": "2019-01-29T19:24:35.000Z",
     "statusChanged": "2019-01-29T19:24:35.000Z",
     "lastLogin": null,
     "lastUpdated": "2019-01-29T19:24:35.000Z",
     "passwordChanged": null,
     "profile": {
       "firstName": "FIRST",
       "lastName": "LAST",
       "mobilePhone": null,
       "migration_app": "true",
       "secondEmail": null,
       "login": "FIRST.LAST@DOMAIN.com",
       "isPasswordInOkta": false,
       "email": "FIRST.LAST@DOMAIN.com"
     },
     "credentials": {
       "provider": {
         "type": "OKTA",
         "name": "OKTA"
       }
     }
   }
   ```

6. The newly created users will be created in a PROVISIONED state with "isPassswordInOkta" set to false and via the group rule automatically be assigned the ODIC web application.

## 2. Create a Custom Login Page

On a Windows server install the IIS server role as well as ASP..Net 4.6.1 and .Net 4.71. Additionally if you plan to edit the project you will need Visual Studio Community 2017, which is free.

![Okta Windows Server Roles Wizard](https://github.com/oktadeveloper/okta-user-migration-dotnet/raw/master/assets/okta-windows-server-roles-wizard.png)

Once IIS is installed, create a new website and application pool called "LDAP Migration." Place the compiled folder structure in the default web directory.

![Okta Migration App in IIS](https://github.com/oktadeveloper/okta-user-migration-dotnet/raw/master/assets/okta-migration-app-in-iis.png)

![Okta Migration App Web Directory](https://github.com/oktadeveloper/okta-user-migration-dotnet/raw/master/assets/okta-migration-app-web-directory.png)

## 3. Configure Variables for Your Tenant

With the web server built and web directory in place, browse to the web directory (`c:\intetpub\wwwroot\LDAPmigration`). Edit web.config and change the following variables:

```
<add key="okta.ApiUrl" value="YOUR_OKTA_TENANT" />
<add key="okta.ApiToken" value="YOUR_OKTA_TENANT_API_TOKEN" />
<!-- use web for sp_int workflow-->
<add key="oidc.spintweb.clientId" value="YOUR_OKTA_OIDCAPP_CLIENTID" />
<add key="oidc.spintweb.clientSecret" value="YOUR_OKTA_OIDCAPP_CLIENTSECRET" />
<add key="oidc.spintweb.RedirectUri_Implicit" value="OIDC_APP_REDIRECT_URI" />
<add key="oidc.spintweb.RedirectUri_AuthCode" value="OIDC_APP_REDIRECT_URI"/>
<add key="oidc.issuer" value="YOUR_OKTA_TENANT_ISSUER" />
<!-- ldap config-->
<add key="ldap.server" value="YOUR_LDAP_SERVER" />
<add key="ldap.port" value="YOUR_LDAP_SERVER_PORT" />
<add key="ldap.baseDn" value=",YOUR_LDAP_SERVER_BASE_DN" />
```

Once complete, save web.config and restart the IIS server.

You should now have all the pieces in place to perform a login.

1. An Okta tenant for which to migrate your users with an OIDC app
2. Sample users in a PROVISIONED state
3. A web server (IIS) to host your custom login page with all the variables specific to you Okta tenant and LDAP server.

## 5. Login

Now that the uses are created in Okta, in a "provisioned" state and the custom login page is up and running, browse to the custom login page and login as a user. The user will be authenticated against the LDAP server, their password set in Okta and finally be redirected to the OIDC web app.

![Okta Custom Sign-In Page](https://github.com/oktadeveloper/okta-user-migration-dotnet/raw/master/assets/okta-custom-sign-in-page.png)

![Okta Authenticate Into App](https://github.com/oktadeveloper/okta-user-migration-dotnet/raw/master/assets/okta-authenticate-into-app.png)

![Okta LDAP Migration Flow](https://github.com/oktadeveloper/okta-user-migration-dotnet/raw/master/assets/okta-ldap-migration-flow.png)

## Appendix

### Changes to the Project

The base project is enough to get you up and running but if changes to the project are needed they must be made in Visual Studio. Upon completion you must publish the web directory to use the changes. To do so:

1. Right click on the project in the project explorer
2. Select Publish
3. Publish method is defaulted to File System simply press Publish
4. The web directory files will be updated with the new version
