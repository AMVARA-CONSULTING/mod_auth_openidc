[![Build Status](https://travis-ci.org/pingidentity/mod_auth_openidc.svg?branch=master)](https://travis-ci.org/pingidentity/mod_auth_openidc)

mod_auth_openidc
================

*mod_auth_openidc* is an authentication/authorization module for the Apache 2.x
HTTP server that functions as an **OpenID Connect Relying Party**, authenticating users against an
OpenID Connect Provider. It can also function as an **OAuth 2.0 Resource Server**, validating 
OAuth 2.0 access tokens presented by OAuth 2.0 Clients.

Overview
--------

This module enables an Apache 2.x web server to operate as an [OpenID Connect](http://openid.net/specs/openid-connect-core-1_0.html)
*Relying Party* (RP) to an OpenID Connect *Provider* (OP). It authenticates users against an OpenID Connect Provider,
receives user identity information from the OP in a so called ID Token and passes the identity information
(a.k.a. claims) in the ID Token to applications hosted and protected by the Apache web server.

It can also be configured as an OAuth 2.0 *Resource Server* (RS), consuming bearer access tokens and introspecting/validating
them against a token introspection endpoint of an OAuth 2.0 Authorization Server and authorizing the Clients based on the
introspection results.

The protected content and/or applications can be served by the Apache server itself or it can be served from elsewhere
when Apache is configured as a Reverse Proxy in front of the origin server(s).

By default the module sets the `REMOTE_USER` variable to the `id_token` `[sub]` claim, concatenated with the OP's Issuer
identifier (`[sub]@[iss]`). Other `id_token` claims are passed in HTTP headers and/or environment variables together with those
(optionally) obtained from the UserInfo endpoint.

It allows for authorization rules (based on standard Apache `Require` primitives) that can be matched against the set
of claims provided in the `id_token`/ `userinfo` claims.

This module supports all defined OpenID Connect flows, including *Basic Client Profile*, *Implicit Client Profile*,
*Hybrid Flows* and the *Refresh Flow*. It supports connecting to multiple OpenID Connect Providers through reading/writing
provider metadata files in a specified metadata directory.

*mod_auth_openidc* supports the following specifications:
- [OpenID Connect](http://openid.net/specs/openid-connect-core-1_0.html)
- [OpenID Connect Dynamic Client Registration](http://openid.net/specs/openid-connect-registration-1_0.html)
- [OpenID Provider Discovery](http://openid.net/specs/openid-connect-discovery-1_0.html)
- [OAuth 2.0 Form Post Response Mode](http://openid.net/specs/oauth-v2-form-post-response-mode-1_0.html)
- [Proof Key for Code Exchange by OAuth Public Clients](https://tools.ietf.org/html/rfc7636)
- [OpenID Connect Session Management](http://openid.net/specs/openid-connect-session-1_0.html). See the [Wiki](https://github.com/pingidentity/mod_auth_openidc/wiki/Session-Management) for information
on how to configure it.

Alternatively the module can operate as an OAuth 2.0 Resource Server to an OAuth 2.0 Authorization Server,
introspecting/validating bearer Access Tokens conforming to [OAuth 2.0 Token Introspection](https://tools.ietf.org/html/rfc7662) or similar.
The `REMOTE_USER` variable setting, passing claims in HTTP headers and authorization based on `Require` primitives
works in the same way as described for OpenID Connect above. See the [Wiki](https://github.com/pingidentity/mod_auth_openidc/wiki/OAuth-2.0-Resource-Server) for information
on how to configure it.

For an exhaustive description of all configuration options, see the file `auth_openidc.conf`
in this directory. This file can also serve as an include file for `httpd.conf`.

How to Use It  
-------------

### OpenID Connect SSO with Google+ Sign-In

Sample configuration for using Google as your OpenID Connect Provider running on
`www.example.com` and `https://www.example.com/example/redirect_uri` registered
as the *redirect_uri* for the client through the Google API Console. You will also
have to enable the `Google+ API` under `APIs & auth` in the [Google API console](https://console.developers.google.com).

```apache
OIDCProviderMetadataURL https://accounts.google.com/.well-known/openid-configuration
OIDCClientID <your-client-id-administered-through-the-google-api-console>
OIDCClientSecret <your-client-secret-administered-through-the-google-api-console>

OIDCRedirectURI https://www.example.com/example/redirect_uri
OIDCCryptoPassphrase <password>

<Location /example/>
   AuthType openid-connect
   Require valid-user
</Location>
```

Note if you want to securely restrict logins to a specific Google Apps domain you would not only
add the `hd=<your-domain>` setting to the `OIDCAuthRequestParams` primitive for skipping the Google Account
Chooser screen, but you must also ask for the `email` scope using `OIDCScope` and use a `Require claim`
authorization setting in the `Location` primitive similar to:

```apache
OIDCScope "openid email"
Require claim hd:<your-domain>
```

The above is an authorization example of an exact match of a provided claim against a string value.
For more authorization options see the [Wiki page on Authorization](https://github.com/pingidentity/mod_auth_openidc/wiki/Authorization).

### OpenID Connect SSO with multiple OpenID Connect Providers

Sample configuration for multiple OpenID Connect providers, which triggers OpenID
Connect Discovery first to find the user's OP.

`OIDCMetadataDir` points to a directory that contains files that contain per-provider
configuration data.

```apache
OIDCMetadataDir <somewhere-writable-for-the-apache-process>/metadata

OIDCRedirectURI https://www.example.com/example/redirect_uri/
OIDCCryptoPassphrase <password>

<Location /example/>
   AuthType openid-connect
   Require valid-user
</Location>
```

For details on configuring multiple providers see https://github.com/pingidentity/mod_auth_openidc/wiki/Multiple-Providers.

### OpenID Connect SSO & OAuth 2.0 Access Control with PingFederate

Another example config for using PingFederate as your OpenID Connect OP and/or
OAuth 2.0 Authorization server, based on the OAuth 2.0 PlayGround 3.x default
configuration and doing claims-based authorization. (running on `localhost` and
`https://localhost/example/redirect_uri/` registered as *redirect_uri* for the
client `ac_oic_client`)

```apache
OIDCProviderMetadataURL https://macbook:9031/.well-known/openid-configuration

OIDCSSLValidateServer Off
OIDCClientID ac_oic_client
OIDCClientSecret abc123DEFghijklmnop4567rstuvwxyzZYXWUT8910SRQPOnmlijhoauthplaygroundapplication

OIDCRedirectURI https://localhost/example/redirect_uri/
OIDCCryptoPassphrase <password>
OIDCScope "openid email profile"

OIDCOAuthIntrospectionEndpoint https://macbook:9031/as/token.oauth2
OIDCOAuthIntrospectionEndpointParams grant_type=urn%3Apingidentity.com%3Aoauth2%3Agrant_type%3Avalidate_bearer
OIDCOAuthIntrospectionEndpointAuth client_secret_basic
OIDCOAuthRemoteUserClaim Username
	
OIDCOAuthSSLValidateServer Off
OIDCOAuthClientID rs_client
OIDCOAuthClientSecret 2Federate

<Location /example/>
   AuthType openid-connect
   #Require valid-user
   Require claim sub:joe
</Location>

<Location /example-api>
   AuthType oauth20
   #Require valid-user
   Require claim Username:joe
   #Require claim scope~\bprofile\b
</Location>
```

Support
-------

See the Wiki pages with Frequently Asked Questions at:  
  https://github.com/pingidentity/mod_auth_openidc/wiki   
There is a Google Group/mailing list at:  
  [mod_auth_openidc@googlegroups.com](mailto:mod_auth_openidc@googlegroups.com)  
The corresponding forum/archive is at:  
  https://groups.google.com/forum/#!forum/mod_auth_openidc  
For commercial support and consultancy you can contact:  
  [info@zmartzone.eu](mailto:info@zmartzone.eu)  

Any questions/issues should go to the mailing list or the
primary author [hans.zandbelt@zmartzone.eu](mailto:hans.zandbelt@zmartzone.eu).  
The Github issues tracker should be used only for bugs reports and feature requests.

Disclaimer
----------

*This software is open sourced by Ping Identity but not supported commercially
by Ping Identity, see also the DISCLAIMER file in this directory. For commercial support
you can contact [ZmartZone IAM](https://www.zmartzone.eu) as described above.*
