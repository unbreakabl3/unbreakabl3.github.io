---
title: VMware vRA + Keycloak (using SAML) + LDAP
date: 2024-02-20
categories: [VMware, Keycloak, vIDM, vRA]
tags: [VMware, vIDM, Keycloak, vRA, Aria Automation]
---

## Keycloak configuration

Let's explore integrating Keycloak (backed by Open LDAP) as an Identity Provider for VMware Identity Manager.

- Login into VMware Identity Manager (IDM) > Administration Console > Identity & Access Management > Identity Providers > Add Identity Provider > Create Third Party IDP

![img-description](/assets/img/vmware-with-keyloak/1*fYsIjKcM_7ncEuObe2ADuA.png){: .shadow }
_VMware IDM_

- Scroll down and download the Service Provider (SP) Metadata XML file. This metadata includes all the necessary information to create a new client in Keycloak.

![img-description](/assets/img/vmware-with-keyloak/1*2KdtwEv-DH_enVgoBampfQ.png){: .shadow }
_VMware IDM_

Now, let us proceed to access Keycloak and initiate the creation of a new client.

- Keycloak > Realm > Clients > Create > Import > select the saved XML metadata we downloaded previously > Save

![img-description](/assets/img/vmware-with-keyloak/1*HPnRsPA1nZob8L9DIu5WXQ.png){: .shadow }
_Keycloak_

- Enable / Disable the client options following the screenshots and save the changes

> We’re using the `username` as a Name ID Format, but it can be changed to any other attribute like `email`
{: .prompt-info }

![img-description](/assets/img/vmware-with-keyloak/1*ZLSAF33jHVQ7kIESRYF9OQ.png){: .shadow }
_Keycloak_

![img-description](/assets/img/vmware-with-keyloak/1*QIqmomYKrmXHCVyTrv3PjA.png){: .shadow }
_Keycloak_

Now let's create 4 mappers: email, givenName, userName, and surname.

- Keycloak > Realm > Clients > our IDM client > Mappers > Create
  - `email` mapper

![img-description](/assets/img/vmware-with-keyloak/1*5_Y7nXYweIcjLDULzfStlw.png){: .shadow }
_Keycloak_

- `givenName` mapper

![img-description](/assets/img/vmware-with-keyloak/1*1FpMZPhK57bQka3HsFS4Lg.png){: .shadow }
_Keycloak_

- `username` mapper

![img-description](/assets/img/vmware-with-keyloak/1*dILdpBgsYB9mdQS_-SxdyA.png){: .shadow }
_Keycloak_

- `surname` mapper

![img-description](/assets/img/vmware-with-keyloak/1*XRkbrlqv0O8dBy9dnubGrw.png){: .shadow }
_Keycloak_

To use a `username` attribute as a login name, we need to make sure this attribute is mapped in the LDAP Federation as well.

- Keycloak > User Federation > your federation > Mappers > Create. Create a new mapper called `username` of type `user-attribute-ldap-mapper` following the screenshot below.

![img-description](/assets/img/vmware-with-keyloak/1*iostWxkzAIcjJ7pchEQ4Yg.png){: .shadow }
_Keycloak_

## IDM configuration

To explore this further, let's take care of IDM. We need to create a new Identity Provider and Directories, which will use this IdP.

- Keycloak > Realm > Realm Settings > General > Endpoints > SAML 2.0 Identity Provider Metadata

Copy the URL by clicking on it, or click on the link and copy the URL from the newly opened window with the XML metadata.

![img-description](/assets/img/vmware-with-keyloak/1*BytLHNkX5g9PHrjfSpndBA.png){: .shadow }
_Keycloak_

The link should look like this: `https://SSO.DOMAIN/realms/REALM-NAME/protocol/saml/descriptor`

- Login into VMware Identity Manager (IDM) > Administration Console > Identity & Access Management > Identity Providers > Add Identity Provider > Create Third Party IDP
  - provide the IdP name (can be any name)
  - paste the SSO Metadata URL into the SAML Metadata window and click on Proceed IdP Metadata. Remove all Name ID formats except the `email` and/or `userName` (this should allow us to log into vRA with LDAP username or email attributes).

![img-description](/assets/img/vmware-with-keyloak/1*qUcyuJzcUE8gtGza4yPCRg.png){: .shadow }
_VMware IDM_

- Name ID format for `email`

![img-description](/assets/img/vmware-with-keyloak/1*g12tRtJgIWMYumNQXAabuQ.png){: .shadow }
_VMware IDM_

- Name ID format for `username`

![img-description](/assets/img/vmware-with-keyloak/1*h-nGHlayGlwFBDkc9PLZ8A.png){: .shadow }
_VMware IDM_

- Enable the Just-in-Time (JIT) provisioning feature and specify any name you want as a Directory Name (this is a display name) and a real domain name in the Domain (can support multiple domains). This will be shown to the user during the login process.
  > Think of JIT as a handy shortcut for user accounts. Instead of pre-creating them, your system magically pulls them out of thin air on a user's first login. Here's the catch: these accounts aren't actual users like ones synchronized from LDAP/AD. They're more like placeholder bookmarks.
  > Each time someone logs in, the system checks for their bookmark. If it doesn't exist, it whips one up instantly. Then, depending on your setup, it either:
  > - **Redirects to Keycloak**: This tells Keycloak to verify the user's identity directly, essentially outsourcing the work.
  > - **Checks LDAP/AD**: It peeks into your existing directory to confirm the username and password match.
  > The key here is no extra data gets copied or stored in your system. Everything remains in its original location, be it Keycloak, LDAP, or AD. This keeps things light and secure.
  {: .prompt-info }

![img-description](/assets/img/vmware-with-keyloak/1*qYbECtbZ5ZM0Y_rjWtujhQ.png){: .shadow }
_VMware IDM_

- Select ALL RANGES (this allows access to IDM from all IPs) or specify the IP range
- Provide an Authentication Method name (it can be any name) and SAML context from the screenshot below

![img-description](/assets/img/vmware-with-keyloak/1*uodZB-SSbviHpJ2Bh-CDxw.png){: .shadow }
_VMware IDM_

![img-description](/assets/img/vmware-with-keyloak/1*N6XpPHm60-6uxnEIQopxpQ.png){: .shadow }
_VMware IDM_

- Save the changes

Let’s check if we have a new directory.

- IDM > Administration Console > Identity & Access Management > Directories. There, we should see a new directory of the type Just-In-Time

![img-description](/assets/img/vmware-with-keyloak/1*gJY_9dQTc_H0bmnezBFLYw.png){: .shadow }
_VMware IDM_

The IDM uses policies to control the authentication way. Let's update it to use our brand-new IdP.

- IDM > Administration Console > Identity & Access Management > Policies

There is an option to edit a default policy or create a new one. For testing purposes, we’ll revise the default policy.

![img-description](/assets/img/vmware-with-keyloak/1*AB2tNafRVD0X2FpQEN4XJA.png){: .shadow }
_VMware IDM_

Because we’re using the browser to authenticate users, let's edit the first policy rule.

![img-description](/assets/img/vmware-with-keyloak/1*ANSLj6Q687z8oC62mTmyrA.png){: .shadow }
_VMware IDM_

Here, we need to select our SSO as a fallback authentication method.

![img-description](/assets/img/vmware-with-keyloak/1*XiuH3eh8bjUBDXXXd3sjsg.png){: .shadow }
_VMware IDM_

> SSO can be set as a primary authentication method. Still, in case of any issue with Keycloak, users will see the error page, and the IDM administrative console will not be available for access because everything will be redirected to SSO first. Just keep it in mind.
> PS. There is a backdoor if you can't log in to IDM with an AD/LDAP user. Go to `https://idm/SAAS/login/0`
{: .prompt-info }

## Export certificate

The last thing we need to do — certificates. SAML is all about certs. Therefore, we must add Keycloak’s certificate to the IDM’s Trusted CA store.

- Go to Keycloak  `https://sso.domain`
  - Obtain Keycloak’s certificate from the browser and save it to the file

![img-description](/assets/img/vmware-with-keyloak/1*2-OAkRbmvVEKY4q6Ucx-aQ.png){: .shadow }
_Keycloak_

- Select Base-64 encoding

![img-description](/assets/img/vmware-with-keyloak/1*IDNnkIcTikxGHqBWWgtYcQ.png){: .shadow }
_Keycloak_

- Open the saved certificate with any text editor and copy and paste it into the IDM Trusted CAs store.

## Add certificate

Building on what we've discussed, let's add the certificate to the Identity Manager.

- IDM > Administration Console > Appliance Settings> Manage Configuration > Install SSL Certificates > Trusted CAs

![img-description](/assets/img/vmware-with-keyloak/1*_3tiEC0H_COGt8bh8OFFPQ.png){: .shadow }
_VMware IDM_

![img-description](/assets/img/vmware-with-keyloak/1*13O9kWoGVEYk465lPnQZVg.png){: .shadow }
_VMware IDM_

## Login in

It is time to test our setup. Go to the vRA login page and click on GO TO LOGIN PAGE.

![img-description](/assets/img/vmware-with-keyloak/1*Jm42V7O4Vj_YUvcbTCyIIg.png){: .shadow }

vRA will redirect us to the IDM login page, proposing we use a built-in System Domain directory and our new SSO.DOMAIN directory

![img-description](/assets/img/vmware-with-keyloak/1*NE8sBTU2IkH9Porv9C5BPg.png){: .shadow }

Select `SSO.DOMAIN` from the dropdown menu, and IDM will redirect us to the Keycloak login page.

![img-description](/assets/img/vmware-with-keyloak/1*h9bwiCwZOfKyw0T-LR6bdw.png){: .shadow }

Provide the username and password, and if everything was done correctly, Keycloak should authenticate the user and redirect them back to vRA.
