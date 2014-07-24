Authentication and Authorization in Kubernetes
==============================================
This document describes the proposed plan for adding authentication and authorization to Kubernetes.  It describes the principals we are adopting/implementing from OAuth 2.0 framework.  It also introduces the new resources to be added to Kubernetes for the purposes of authentication and access control of existing and new resources in Kubernetes.  

OAuth Terminology
-----------------
  Copied over from [OAuth-2.0-section-1.1](http://tools.ietf.org/html/rfc6749#section-1.1)

  **resource owner**  
  An entity capable of granting access to a protected resource.
  When the resource owner is a person, it is referred to as an
  end-user.

  **client**  
  An application making protected resource requests on behalf of the
  resource owner and with its authorization.  The term "client" does
  not imply any particular implementation characteristics (e.g.,
  whether the application executes on a server, a desktop, or other
  devices).

  **resource server**  
  The server hosting the protected resources, capable of accepting
  and responding to protected resource requests using access tokens.

  **authorization server**  
  The server issuing access tokens to the client after successfully
  authenticating the resource owner and obtaining authorization.

  ***NOTE: In the proposed "default" implementation of OAuth in Kubernetes the 
  Authorization Server and Resource Server are one of the same.***

  **Authorization code**  
  Condensed and copied from [OAuth-2.0-section-1.3.1](http://tools.ietf.org/html/rfc6749#section-1.3.1)  

  An authorization code is a credential representing the resource
  owner's authorization (to access its protected resources) used by the
  client to obtain an access token. 

  **Access token**  
  Condensed and copied from [OAuth-2.0-section-1.4](http://tools.ietf.org/html/rfc6749#section-1.4)  

  Access tokens are credentials used to access protected resources.  An
  access token is a string representing an authorization issued to the
  client.  The string is usually opaque to the client.  Tokens
  represent specific scopes and duration of access, granted by the
  resource owner, and enforced by the resource server and authorization
  server. The access token provides an abstraction layer, replacing 
  different authorization constructs (e.g., username and password) with
  a single token understood by the resource server.  This abstraction enables
  issuing access tokens more restrictive than the authorization grant
  used to obtain them, as well as removing the resource server's need
  to understand a wide range of authentication methods.


Authorization Grant
-------------------
   The OAuth specification defines four grant types: 
   
   - Authorization Code (See [OAuth-2.0-section-1.3.1](http://tools.ietf.org/html/rfc6749#section-1.3.1))
   - Implicit (See [OAuth-2.0-section-1.3.2](http://tools.ietf.org/html/rfc6749#section-1.3.2))
   - Resource Owner Password Credentials (See [OAuth-2.0-section-1.3.3](http://tools.ietf.org/html/rfc6749#section-1.3.3))
   - Client Credentials (See [OAuth-2.0-section-1.3.4](http://tools.ietf.org/html/rfc6749#section-1.3.4))
   
   It also provides an extension mechanism for defining additional grant types.

   ***Note, in this document we are only going to cover grant types:***
   
   - resource owner password credentials
   - authorization code


Resource Owner Password Credentials
-----------------------------------
   Copied over from [OAuth-2.0-section-4.3](http://tools.ietf.org/html/rfc6749#section-4.3)

   The resource owner password credentials (i.e., username and password)
   can be used directly as an authorization grant to obtain an access
   token.

   Even though this grant type requires direct client access to the
   resource owner credentials, the resource owner credentials are used
   for a single request and are exchanged for an access token.  This
   grant type can eliminate the need for the client to store the
   resource owner credentials for future use, by exchanging the
   credentials with a long-lived access token or refresh token.

     +----------+
     | Resource |
     |  Owner   |
     |          |
     +----------+
          v
          |    Resource Owner
         (A) Password Credentials
          |
          v
     +---------+                                  +---------------+
     |         |>--(B)---- Resource Owner ------->|               |
     |         |         Password Credentials     | Authorization |
     | Client  |                                  |     Server    |
     |         |<--(C)---- Access Token ---------<|               |
     |         |    (w/ Optional Refresh Token)   |               |
     +---------+                                  +---------------+

            Resource Owner Password Credentials Flow

   The flow illustrated includes the following steps:

   (A)  The resource owner provides the client with its username and
        password.

   (B)  The client requests an access token from the authorization
        server's token endpoint by including the credentials received
        from the resource owner.  When making the request, the client
        authenticates with the authorization server.

   (C)  The authorization server authenticates the client and validates
        the resource owner credentials, and if valid, issues an access
        token.


Authorization Code
------------------
   Copied over from [OAuth-2.0-section-4.1](http://tools.ietf.org/html/rfc6749#section-4.1)

   The authorization code is obtained by using an authorization server
   as an intermediary between the client and resource owner.  Instead of
   requesting authorization directly from the resource owner, the client
   directs the resource owner to an authorization server, which in turn 
   directs the resource owner back to the client with the authorization 
   code.

   Before directing the resource owner back to the client with the
   authorization code, the authorization server authenticates the
   resource owner and obtains authorization.  Because the resource owner
   only authenticates with the authorization server, the resource
   owner's credentials are never shared with the client.

   The authorization code provides a few important security benefits,
   such as the ability to authenticate the client, as well as the
   transmission of the access token directly to the client without
   passing it through the resource owner's user-agent and potentially
   exposing it to others, including the resource owner.

   The authorization code grant type is used to obtain both access
   tokens and refresh tokens and is optimized for confidential clients.
   Since this is a redirection-based flow, the client must be capable of
   interacting with the resource owner's user-agent (typically a web
   browser) and capable of receiving incoming requests (via redirection)
   from the authorization server.

     +----------+
     | Resource |
     |   Owner  |
     |          |
     +----------+
          ^
          |
         (B)
     +----|-----+          Client Identifier      +---------------+
     |         -+----(A)-- & Redirection URI ---->|               |
     |  User-   |                                 | Authorization |
     |  Agent  -+----(B)-- User authenticates --->|     Server    |
     |          |                                 |               |
     |         -+----(C)-- Authorization Code ---<|               |
     +-|----|---+                                 +---------------+
       |    |                                         ^      v
      (A)  (C)                                        |      |
       |    |                                         |      |
       ^    v                                         |      |
     +---------+                                      |      |
     |         |>---(D)-- Authorization Code ---------'      |
     |  Client |          & Redirection URI                  |
     |         |                                             |
     |         |<---(E)----- Access Token -------------------'
     +---------+       (w/ Optional Refresh Token)

                     Authorization Code Flow

   The flow illustrated includes the following steps:

   (A)  The client initiates the flow by directing the resource owner's
        user-agent to the authorization endpoint.  The client includes
        its client identifier, requested scope, local state, and a
        redirection URI to which the authorization server will send the
        user-agent back once access is granted (or denied).

   (B)  The authorization server authenticates the resource owner (via
        the user-agent) and establishes whether the resource owner
        grants or denies the client's access request.

   (C)  Assuming the resource owner grants access, the authorization
        server redirects the user-agent back to the client using the
        redirection URI provided earlier (in the request or during
        client registration).  The redirection URI includes an
        authorization code and any local state provided by the client
        earlier.

   (D)  The client requests an access token from the authorization
        server's token endpoint by including the authorization code
        received in the previous step.  When making the request, the
        client authenticates with the authorization server.  The client
        includes the redirection URI used to obtain the authorization
        code for verification.

   (E)  The authorization server authenticates the client, validates the
        authorization code, and ensures that the redirection URI
        received matches the URI used to redirect the client in
        step (C).  If valid, the authorization server responds back with
        an access token and, optionally, a refresh token.


Protocol Endpoints
------------------

   The authorization process utilizes two authorization server endpoints
   (HTTP resources):

   -  Authorization endpoint - used by the client to obtain
      authorization from the resource owner via user-agent redirection.

   -  Token endpoint - used by the client to exchange an authorization
      grant for an access token, typically with client authentication.

   For a full list of endpoints not covered in this document see [OAuth-2.0-section-3](http://tools.ietf.org/html/rfc6749#section-3)
 
Authorization Endpoint
----------------------
   The authorization endpoint is used to interact with the resource
   owner and obtain an authorization grant.  The authorization server
   MUST first verify the identity of the resource owner. 

   See more info [OAuth-2.0-section-3.1](http://tools.ietf.org/html/rfc6749#section-3.1)

Token Endpoint
--------------

   The token endpoint is used by the client to obtain an access token by
   presenting its authorization grant or refresh token.  The token
   endpoint is used with every authorization grant except for the
   implicit grant type (since an access token is issued directly).

   See more info [OAuth-2.0-section-3.2](http://tools.ietf.org/html/rfc6749#section-3.2)

Authorization Request
---------------------
   Compiled from combining [OAuth-2.0-section-4.1.1](http://tools.ietf.org/html/rfc6749#section-4.1.1) and [OAuth-2.0-section-4.2.1](http://tools.ietf.org/html/rfc6749#section-4.2.1)

   The client constructs the request URI by adding the following
   parameters to the query component of the authorization endpoint URI
   using the "application/x-www-form-urlencoded" format:

   **response_type**  
   REQUIRED Value MUST be set to "code" for "Authorization Code" 
   grant and "token" for "Implicit" grant (not covered id this doc)

   **client_id**  
   REQUIRED The client identifier

   **redirect_uri**  
   OPTIONAL The redirection endpoint URI MUST be an absolute URI

   **scope**  
   OPTIONAL The scope of the access request.

   **state**  
   RECOMMENDED An opaque value used by the client to maintain
   state between the request and callback.  The authorization
   server includes this value when redirecting the user-agent back
   to the client.  The parameter SHOULD be used for preventing
   cross-site request forgery as described in Section 10.12.

   The client directs the resource owner to the constructed URI using an
   HTTP redirection response, or by other means available to it via the
   user-agent.

   For example, the client directs the user-agent to make the following
   HTTP request using TLS:

    GET /authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz
        &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb HTTP/1.1
    Host: server.example.com

   The authorization server validates the request to ensure that all
   required parameters are present and valid.  If the request is valid,
   the authorization server authenticates the resource owner and directs 
   the user-agent to the provided client redirection URI using an HTTP
   redirection response.

Authorization Response
----------------------
   Copied over from [OAuth-2.0-section-4.1.2](http://tools.ietf.org/html/rfc6749#section-4.1.2)

   If the resource owner grants the access request, the authorization
   server issues an authorization code and delivers it to the client by
   adding the following parameters to the query component of the
   redirection URI using the "application/x-www-form-urlencoded" format:

   **code**  
   REQUIRED The authorization code generated by the
   authorization server.  The authorization code MUST expire
   shortly after it is issued to mitigate the risk of leaks.  A
   maximum authorization code lifetime of 10 minutes is RECOMMENDED.
   The client MUST NOT use the authorization code
   more than once.  If an authorization code is used more than
   once, the authorization server MUST deny the request and SHOULD
   revoke (when possible) all tokens previously issued based on
   that authorization code.  The authorization code is bound to
   the client identifier and redirection URI.

   **state**   
   REQUIRED if the "state" parameter was present in the client
   authorization request.  The exact value received from the
   client.

   For example, the authorization server redirects the user-agent by
   sending the following HTTP response:

     HTTP/1.1 302 Found
     Location: https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA
               &state=xyz

   The client MUST ignore unrecognized response parameters.  The
   authorization code string size is left undefined by this
   specification.  The client should avoid making assumptions about code
   value sizes.  The authorization server SHOULD document the size of
   any value it issues.

Authorization Request Error Response
------------------------------------
   Copied over from [OAuth-2.0-section-4.1.2.1](http://tools.ietf.org/html/rfc6749#section-4.1.2.1)
   
   If the request fails due to a missing, invalid, or mismatching
   redirection URI, or if the client identifier is missing or invalid,
   the authorization server SHOULD inform the resource owner of the
   error and MUST NOT automatically redirect the user-agent to the
   invalid redirection URI.

   If the resource owner denies the access request or if the request
   fails for reasons other than a missing or invalid redirection URI,
   the authorization server informs the client by adding the following
   parameters to the query component of the redirection URI using the
   "application/x-www-form-urlencoded" format:

   **error**  
   REQUIRED A single ASCII [USASCII] error code from the following:

   _invalid_request_  
        The request is missing a required parameter, includes an
        invalid parameter value, includes a parameter more than
        once, or is otherwise malformed.

   _unauthorized_client_  
        The client is not authorized to request an authorization
        code using this method.

   _access_denied_  
        The resource owner or authorization server denied the
        request.

   _unsupported_response_type_  
        The authorization server does not support obtaining an
        authorization code using this method.

   _invalid_scope_  
        The requested scope is invalid, unknown, or malformed.

   _server_error_  
        The authorization server encountered an unexpected
        condition that prevented it from fulfilling the request.
        (This error code is needed because a 500 Internal Server
        Error HTTP status code cannot be returned to the client
        via an HTTP redirect.)

   _temporarily_unavailable_  
        The authorization server is currently unable to handle
        the request due to a temporary overloading or maintenance
        of the server.  (This error code is needed because a 503
        Service Unavailable HTTP status code cannot be returned
        to the client via an HTTP redirect.)

   Values for the "error" parameter MUST NOT include characters
   outside the set %x20-21 / %x23-5B / %x5D-7E.

   **error_description**  
         OPTIONAL.  Human-readable ASCII [USASCII] text providing
         additional information, used to assist the client developer in
         understanding the error that occurred.
         Values for the "error_description" parameter MUST NOT include
         characters outside the set %x20-21 / %x23-5B / %x5D-7E.

   **error_uri**  
         OPTIONAL.  A URI identifying a human-readable web page with
         information about the error, used to provide the client
         developer with additional information about the error.
         Values for the "error_uri" parameter MUST conform to the
         URI-reference syntax and thus MUST NOT include characters
         outside the set %x21 / %x23-5B / %x5D-7E.

   **state**  
         REQUIRED if a "state" parameter was present in the client
         authorization request.  The exact value received from the
         client.

   For example, the authorization server redirects the user-agent by
   sending the following HTTP response:

   HTTP/1.1 302 Found
   Location: https://client.example.com/cb?error=access_denied&state=xyz


Access Token Request
--------------------
   Compiled from combining [OAuth-2.0-section-4.1.3](http://tools.ietf.org/html/rfc6749#section-4.1.3) and [OAuth-2.0-section-4.3.2](http://tools.ietf.org/html/rfc6749#section-4.3.2)

   The client makes a request to the token endpoint by sending the
   following parameters using the "application/x-www-form-urlencoded"
   format with a character encoding of UTF-8 in the HTTP request 
   entity-body:

   **grant_type**  
   REQUIRED Value MUST be set to "authorization_code" OR "password" depending on authorization type.

   **code**  
   REQUIRED, if grant type is "authorization_code"

   **username**  
   REQUIRED, if grant type is "password".

   **password**  
   REQUIRED, if grant type is "password".

   **redirect_uri**  
   REQUIRED, if the "redirect_uri" parameter was included in the
   authorization request and their values MUST be identical. Not
   applicable for "password" grant types.
   
   **client_id**  
   REQUIRED, if the client is not authenticating with the
   authorization server. Not applicable for "password" grant types.

   In case of grant_type=password, the client makes the following HTTP request:

     POST /token HTTP/1.1
     Host: server.example.com
     Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
     Content-Type: application/x-www-form-urlencoded

     grant_type=password&username=johndoe&password=A3ddj3w


   In case of grant_type=authorization_code, the client makes the following 
   HTTP request:

     POST /token HTTP/1.1
     Host: server.example.com
     Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
     Content-Type: application/x-www-form-urlencoded

     grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA
     &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb

   The authorization server MUST:

   o  require client authentication for confidential clients or for any
      client that was issued client credentials (or with other
      authentication requirements),

   o  authenticate the client if client authentication is included,

   o  ensure that the authorization code was issued to the authenticated
      confidential client, or if the client is public, ensure that the
      code was issued to "client_id" in the request,

   o  verify that the authorization code is valid, and

   o  ensure that the "redirect_uri" parameter is present if the
      "redirect_uri" parameter was included in the initial authorization
      request as described in Section 4.1.1, and if included ensure that
      their values are identical.

Access Token Response
---------------------
   Compiled from combining [OAuth-2.0-section-4.1.4](http://tools.ietf.org/html/rfc6749#section-4.1.4) and [OAuth-2.0-section-4.3.3](http://tools.ietf.org/html/rfc6749#section-4.3.3)

   The authorization server issues an access token and optional refresh
   token, and constructs the response by adding the following parameters
   to the entity-body of the HTTP response with a 200 (OK) status code:

   **access_token**  
     REQUIRED.  The access token issued by the authorization server.

   **token_type**  
     REQUIRED.  The type of the token issued. Value is case insensitive.

   **expires_in**  
     RECOMMENDED.  The lifetime in seconds of the access token.  For
     example, the value "3600" denotes that the access token will
     expire in one hour from the time the response was generated.
     If omitted, the authorization server SHOULD provide the
     expiration time via other means or document the default value.

   **refresh_token**  
     OPTIONAL.  The refresh token, which can be used to obtain new
     access tokens using the same authorization grant.

   **scope**  
     OPTIONAL, if identical to the scope requested by the client;
     otherwise, REQUIRED.

   The parameters are included in the entity-body of the HTTP response
   using the "application/json" media type.

   The authorization server MUST include the HTTP "Cache-Control"
   response header field with a value of "no-store" in any response 
   containing tokens, credentials, or other sensitive information, as 
   well as the "Pragma" response header field with a value of "no-cache".

   For example:

     HTTP/1.1 200 OK
     Content-Type: application/json;charset=UTF-8
     Cache-Control: no-store
     Pragma: no-cache

     {
       "access_token":"2YotnFZFEjr1zCsicMWpAA",
       "token_type":"example",
       "expires_in":3600,
       "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
       "example_parameter":"example_value"
     }

   The client MUST ignore unrecognized value names in the response.  The
   sizes of tokens and other values received from the authorization
   server are left undefined.  The client should avoid making
   assumptions about value sizes.  The authorization server SHOULD
   document the size of any value it issues.


Access Token Error Response
---------------------------
   Copied over from [OAuth-2.0-section-5.2](http://tools.ietf.org/html/rfc6749#section-5.2)

   The authorization server responds with an HTTP 400 (Bad Request)
   status code (unless specified otherwise) and includes the following
   parameters with the response:

   **error**   
   REQUIRED A single ASCII [USASCII] error code from the following:

   _invalid_request_  
   The request is missing a required parameter, includes an
      unsupported parameter value (other than grant type),
      repeats a parameter, includes multiple credentials,
      utilizes more than one mechanism for authenticating the
      client, or is otherwise malformed.

   _invalid_client_  
   Client authentication failed (e.g., unknown client, no
      client authentication included, or unsupported
      authentication method).  The authorization server MAY
      return an HTTP 401 (Unauthorized) status code to indicate
      which HTTP authentication schemes are supported.  If the
      client attempted to authenticate via the "Authorization"
      request header field, the authorization server MUST
      respond with an HTTP 401 (Unauthorized) status code and
      include the "WWW-Authenticate" response header field
      matching the authentication scheme used by the client.

   _invalid_grant_  
      The provided authorization grant (e.g., authorization
      code, resource owner credentials) or refresh token is
      invalid, expired, revoked, does not match the redirection
      URI used in the authorization request, or was issued to
      another client.

   _unauthorized_client_  
      The authenticated client is not authorized to use this
      authorization grant type.

   _unsupported_grant_type_   
      The authorization grant type is not supported by the
      authorization server.

   _invalid_scope_  
      The requested scope is invalid, unknown, malformed, or
      exceeds the scope granted by the resource owner.

Values for the "error" parameter MUST NOT include characters outside the set %x20-21 / %x23-5B / %x5D-7E.

   **error_description**  
   OPTIONAL.  Human-readable ASCII [USASCII] text providing
   additional information, used to assist the client developer in
   understanding the error that occurred.
   Values for the "error_description" parameter MUST NOT include
   characters outside the set %x20-21 / %x23-5B / %x5D-7E.

   **error_uri**  
   OPTIONAL.  A URI identifying a human-readable web page with
   information about the error, used to provide the client
   developer with additional information about the error.
   Values for the "error_uri" parameter MUST conform to the
   URI-reference syntax and thus MUST NOT include characters
   outside the set %x21 / %x23-5B / %x5D-7E.

   The parameters are included in the entity-body of the HTTP response
   using the "application/json" media type.  The parameters are serialized
   into a JSON structure by adding each parameter at the highest structure
   level.  Parameter names and string values are included as JSON strings.  
   Numerical values are included as JSON numbers.  The order of parameters 
   does not matter and can vary.

   For example:

     HTTP/1.1 400 Bad Request
     Content-Type: application/json;charset=UTF-8
     Cache-Control: no-store
     Pragma: no-cache

     {
       "error":"invalid_request"
     }


New Resources
-------------

**User**

This resource represents the end-user of the Kubernetes API. It will be stored in etcd along with the existing Kubernetes resources i.e. pods, replication controllers, etc.

A new user resource can be created via POST /users by the end-user (if self-registration is enabled) or by an administrator.

_Example_

         Request: POST /users
         {
           "username": "jdoe",
           "email": "jdoe@test.org",
           "password": "secret123",
           "role" : "developer",
           "entitlements": ["create_project", "create_template", "create_pod"]
         }

The following parameters are optional:
- password - a temporary password will be generated if not provided.
- role - if not specified the user will be assigned a default role (if specified) or null.
- entitlements - defaults to null.


         Response: HTTP/1.1 201 CREATED
         {
           "id": "53c4249f076573c0f4000001",
           "username": "jdoe",
           "password": "temporary123",
           "email":  "jdoe@test.org",
           "role": "developer",
           "entitlements": ["create_project", "create_template"]
         }
           
 
**Group**

This resource enables grouping of end-users into one entity for ease of assigning access control. A group resource can be created by POST /groups

_Example_

         Request: POST /groups
         {
           "name": "engineering",
           "users": ["63c4249f088573c0f4000090", "54a4249f076573c0f4000022"]
         }

The users parameter is optional and can be updates via PUT requests.


         Response: HTTP/1.1 201 CREATED
         {
           "id": "53c4249f076573c0f4000001",
           "name": "engineering",
           "users": ["63c4249f088573c0f4000090", "54a4249f076573c0f4000022"]
           "owner": "53c4249f076573c0f4000001"
         }


**Role**

This resource represents a set of capabilites that can be assigned to a user or group as part of access control assignment. 

_Example_

         Request: POST /roles
         {
           "name": "developer",
           "entitlements": ["create_templates", "create_images"]
         }

         Response: HTTP/1.1 201 CREATED
         {
           "id": "53c4249f076573c0f4000023",
           "name": "developer",
           "entitlements": ["create_templates", "create_image_streams"]
           "owner": "53c4249f076573c0f4000001"
         }

**Access Token and Authorization**

We have already described these new resources and their endpoints in the previous section. 


Granting access to resources
============================

Implict grants
--------------
As the owner of a resource by default the user posses all the rights to that resource including giving authorization to other users or groups. The ownership of a resource is determined by the field "owner" stored in the resource itself. 

_Example_

         {
           "name": "engineering",
           "users": ["63c4249f088573c0f4000090", "54a4249f076573c0f4000022"]
           "owner": "53c4249f076573c0f4000001"
         }

Another type of implicit grant worth mentioning is by viture of parent child relationship.  For example if a user has viewing rights on a project, they will have viewing right to the related resources like services, pod, etc.

Explicit grants
---------------
A resource owner can authorize other users or groups by way of adding them to the resources accessible_to field. For example a user can give access to a project they created to other users. 

_Example_

      Request: PUT /projects/id
         {
           "accessible_to": [
                              { 
				"principal": "53c4249f076573c0f4000001",
                                "type": "user",
	                        "role": "53c4249f076573c0f4000023",
                                "entitlements": ["update", "delete"]
                              },
                              { 
				"principal": "54c4249f076573c0f4000022",
                                "type": "group",
                                "entitlements": ["view"]
                              },
                              { 
				"principal": "53c4249f076573c0f4000001",
                                "type": "user",
                                "role": ["admin"]
                              }
                            ]
         }

The following parameters are optional:
- role - defaults to null
- entitlements - defaults to null

However, for the access grant to make sense one or both need to be defined. The entitlements parameters can be used by itself or to add entitlements above and beyond granted by the role.





