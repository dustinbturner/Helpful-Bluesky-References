OAuth
The OAuth profile for atproto is new and may be revised based on feedback from the development community and ongoing standards work. Read more about the rollout in the OAuth Roadmap.

This specification is authoritative, but is not an implementation guide and does not provide much background or context around design decisions. The earlier design proposal is not authoritative but provides more context and examples. SDK documentation and the client implementation guide are more approachable for developers.

OAuth is the primary mechanism in atproto for clients to make authorized requests to PDS instances. Most user-facing software is expected to use OAuth, including "front-end" clients like mobile apps, rich browser apps, or native desktop apps, as well as "back-end" clients like web services.

See the HTTP API specification for other forms of auth in atproto, including legacy HTTP client sessions/tokens, and inter-service auth.

OAuth is a constantly evolving framework of standards and best practices, standardized by the IETF. atproto uses a specific "profile" of OAuth which mandates a particular combination of OAuth standards, as described in this document.

At a high level, we start with the "OAuth 2.1" (draft-ietf-oauth-v2-1) evolution of OAuth 2.0, which means:

only the "authorization code" OAuth 2.0 grant type is supported, not "implicit" or other grant types
mandatory Proof Key for Code Exchange (PKCE, RFC 7636)
security best practices (draft-ietf-oauth-security-topics and draft-ietf-oauth-browser-based-apps) are required
Unlike a centralized app platform, in atproto there are many independent server implementations, so server discovery and client registration are automated using a combination of public auth server metadata and public client metadata. The client_id is a fully-qualified web URL pointing to the public client metadata (JSON document). There is no client_secret shared between servers and clients. When initiating a login with a handle or DID, an atproto-specific identity resolution step is required to discover the account’s PDS network location.

In OAuth terminology, an atproto Personal Data Server (PDS) is a "Resource Server" to which authorized HTTP requests are made using access tokens. Sometimes the PDS is also the "Authorization Server" - which services OAuth authorization flows and token requests - while in other situations a separate "entryway" service acts as the Authorization Server for multiple PDS instances. Clients from a metadata file from the PDS to discover the Authorization Server network location.

DPoP (with mandatory server issued nonces) is required to bind auth tokens to specific client software instances (eg, end devices or browser sessions). Pushed Authentication Requests (PAR) are used to streamline the authorization request flow. "Confidential" clients use JWTs signed with a secret key to authenticate the client software to Authorization Servers when making authorization requests.

Automated client registration using client metadata is one of the more novel aspects of OAuth in atproto. As of August 2024, client metadata is still an Internet Draft (draft-parecki-oauth-client-id-metadata-document); it should not be confused with the existing "Dynamic Client Registration" standard (RFC 7591). We are hopeful other open protocols will adopt similar automated registration flows in the future, but there may not be general OAuth ecosystem support for some time.

OAuth 2.0 is traditionally an authorization (authz) system, not an authentication (authn) system, meaning that it is not always a solution for pure account authentication use cases, such as "Signup/Login with XYZ" identity integrations. OpenID Connect (OIDC), which builds on top of OAuth 2.0, is usually the recommended standard for identity authentication. Unfortunately, the current version of OIDC does not enable authentication of atproto identities in a secure and generic way. The atproto profile of OAuth includes a (mandatory) mechanism for account authentication during the authorization flow and can be used for atproto identity authentication use cases.

Clients
This section describes requirements for OAuth clients, which are enforced by Authorization Servers.

OAuth client software is identified by a globally unique client_id. Distinct variants of client software may have distinct client_id values; for example the browser app and Android (mobile OS) variants of the same software might have different client_id values. As required by the draft-parecki-oauth-client-id-metadata-document specification draft, the client_id must be a fully-qualified web URL from which the client-metadata JSON document can be fetched. For example, https://app.example.com/client-metadata.json. Some more about the client_id:

it must be a well-formed URL, following the W3C URL specification
the schema must be https://, and there must not be a port number included. Note that there is a special exception for http://localhost client_id values for development, see details below
the path does not need to include client-metadata.json, but it is helpful convention
Authorization Servers which support both the atproto OAuth profile and other forms of OAuth should take care to prevent client_id value collisions. For example, client_id values for clients which are not auto-registered should never have the prefix https:// or http://.

Types of Clients
All atproto OAuth clients need to meet a core set of standards and requirements, but there are a few variations in capabilities (such as session lifetime) depending on the security properties of the client itself.

As described in the OAuth 2.0 specification (RFC 6749), every client is one of two broad types:

confidential clients are clients which can authenticate themselves to Authorization Servers using a cryptographic signing key. This allows refresh tokens to be bound to the specific client. Note that this form of client authentication is distinct from DPoP: the client authentication key is common to all client sessions (although it can be rotated). This usually means that there is a web service controlled by the client which holds the key. Because they are authenticated and can revoke tokens in a security incident, confidential clients may be trusted with longer session and token lifetimes.
public clients do not authenticate using a client signing key, either because they don’t have a server-side component (the client software all runs on end-user devices), or they simply chose not to implement it.
It is acceptable for a web service to act as a public client, and conversely it is possible for mobile apps and browser apps to coordinate with a token-mediating backend service and for the combination to form a confidential client. Mobile apps and browser apps can also adopt a "backend-for-frontend" (BFF) architecture with a web service backend acting as the OAuth client. This document will use the "public" vs "confidential" client terminology for clarity.

The environment a client runs in also impacts the type of redirect (callback) URLs it uses during the Authorization Flow:

web clients include web services and browser apps. Redirect URLs are regular web URLs which open in a browser.
native clients include some mobile and desktop native clients. Redirect URLs may use platform-specific app callback schemes to open in the app itself.
Authorization Servers may maintain a set of "trusted" clients, identified by client_id. Because any client could use unverified client metadata to impersonate a better-known app or brand, Authorization Servers should not display such metadata to end users in the Authorization Interface by default. Trusted clients can have additional metadata shown, such as a readable name (client_name), project URI (client_uri, which may have a different domain/origin than client_id) and logo (logo_uri). See the "Security Considerations" section for more details.

Clients which are only using atproto OAuth for account authentication (without authorization to access PDS resources) should request minimal scopes (see "Scopes" section), but still need to implement most of the authorization flow. In particular, it is critical that they check the sub field in a token response to verify the account identity (this is an atproto-specific detail).

Client ID Metadata Document
The Client ID Metadata Document specification (draft-parecki-oauth-client-id-metadata-document is still a draft and may evolve over time. Our intention is to evolve and align with subsequent drafts and any final standard, while minimizing disruption and breakage with existing implementations.

Clients must publish a "client metadata" JSON file on the public web. This will be fetched dynamically by Authorization Servers as part of the authorization request (PAR) and at other times during the session lifecycle. The response HTTP status must be 200 (not another 2xx or a redirect), with a JSON object body with the correct Content-Type (application/json).

Authorization Servers need to fetch client metadata documents from the public web. They should use a hardened HTTP client for these requests (see "OAuth Security Considerations"). Servers may cache client metadata responses, optionally respecting HTTP caching headers (within limits). Minimum and maximum cache TTLs are not currently specified, but should be chosen to ensure that auth token requests predicated on stale confidential client authentication keys (jwks or jwks_uris) are rejected in a timely manner.

The following fields are relevant for all client types:

client_id (string, required): the client_id. Must exactly match the full URL used to fetch the client metadata file itself
application_type (string, optional): must be one of web or native, with web as the default if not specified. Note that this is field specified by OpenID/OIDC, which we are borrowing. Used by the Authorization Server to enforce the relevant "best current practices".
grant_types (array of strings, required): authorization_code must always be included. refresh_token is optional, but must be included if the client will make token refresh requests.
scope (string, sub-strings space-separated, required): all scope values which might be requested by this client are declared here. The atproto scope is required, so must be included here. See "Scopes" section.
response_types (array of strings, required): code must be included.
redirect_uris (array of strings, required): at least one redirect URI is required. See Authorization Request Fields section for rules about redirect URIs, which also apply here.
token_endpoint_auth_method (string, optional): confidential clients must set this to private_key_jwt.
token_endpoint_auth_signing_alg (string, optional): none is never allowed here. The current recommended and most-supported algorithm is ES256, but this may evolve over time. Authorization Servers will compare this against their supported algorithms.
dpop_bound_access_tokens (boolean, required): DPoP is mandatory for all clients, so this must be present and true
jwks (object with array of JWKs, optional): confidential clients must supply at least one public key in JWK format for use with JWT client authentication. Either this field or the jwks_uri field must be provided for confidential clients, but not both.
jwks_uri (string, optional): URL pointing to a JWKS JSON object. See jwks above for details.
These fields are optional but recommended:

client_name (string, optional): human-readable name of the client
client_uri (string, optional): not to be confused with client_id, this is a homepage URL for the client. If provided, the client_uri must have the same hostname as client_id.
logo_uri (string, optional): URL to client logo. Only https: URIs are allowed.
tos_uri (string, optional): URL to human-readable terms of service (ToS) for the client. Only https: URIs are allowed.
policy_uri (string, optional): URL to human-readable privacy policy for the client. Only https: URIs are allowed.
See "OAuth Security Considerations" below for when client_name, client_uri, and logo_uri will or will not be displayed to end users.

Additional optional client metadata fields are enumerated with IANA. Note that these are shared with the "Dynamic Client Registration" standard, which is not used directly by the atproto OAuth profile.

Localhost Client Development
When working with a developent environment (Authorization Server and Client), it may be difficult for developers to publish in-progress client metadata at a public URL so that authorization servers can access it. This may even be true for development environments using a containerized Authorization Server and local DNS, because of SSRF protections against local IP ranges.

To make development workflows easier, a special exception is made for clients with client_id having origin http://localhost (with no port number specified). Authorization Servers are encouraged to support this exception - including in production environments - but it is optional.

In a localhost client_id scenario, the Authorization Server should verify that the scheme is http, and that the hostname is exactly localhost with no port specified. IP addresses (127.0.0.1, etc) are not supported. The path parameter must be empty (/).

In the Authorization Request, the redirect_uri must match one of those supplied (or a default). Path components must match, but port numbers are not matched.

Some metadata fields can be configured via query parameter in the client_id URL (with appropriate urlencoding):

redirect_uri (string, multiple query parameters allowed, optional): allows declaring a local redirect/callback URL, with path component matched but port numbers ignored. The default values (if none are supplied) are http://127.0.0.1/ and http://[::1]/.
scope (string with space-separated values, single query parameter allowed, optional): the set of scopes which might be requested by the client. Default is atproto.
The other parameters in the virtual client metadata document will be:

client_id (string): the exact client_id (URL) used to generate the virtual document
client_name (string): a value chosen by the Authorization Server (e.g. "Development client")
response_types (array of strings): must include code
grant_types (array of strings): authorization_code and refresh_token
token_endpoint_auth_method: none
application_type: native
dpop_bound_access_tokens: true
Note that this works as a public client, not a confidential client.

Identity Authentication
As mentioned in the introduction, OAuth 2.0 generally provides only Authorization (authz), and additional standards like OpenID/OIDC are used for Authentication (authn). The atproto profile of OAuth requires authentication of account identity and supports the use case of simple identity authentication without additional resource access authorization.

In atproto, account identity is anchored in the account DID, which is the permanent, globally unique, publicly resolvable identifier for the account. The DID resolves to a DID document which indicates the current PDS host location for the account. That PDS (combined with an optional entryway) is the authorization authority and the OAuth Authorization Server for the account. When speaking to any Authorization Server, it is critical (mandatory) for clients to confirm that it is actually the authoritative server for the account in question, which means independently resolving the account identity (by DID) and confirming that the Authorization Server matches. It is also critical (mandatory) to confirm at the end of an authorization flow that the Authorization Server actually authorized the expected account. The reason this is necessary is to confirm that the Authorization Server is authoritative for the account in question. Otherwise a malicious server could authenticate arbitrary accounts (DIDs) to the client.

Clients can start an auth flow in one of two ways:

starting with a public account identifier, provided by the user: handle or DID
starting with a server hostname, provided by the user: PDS or entryway, mapping to either Resource Server and/or Authorization Server
One use case for starting with a server instead of an account identifier is when the user does not remember their full account handle or only knows their account email. Another is for authentication when a user’s handle is broken. The user will still need to know their hosting provider in these situation.

When starting with an account identifier, the client must resolve the atproto identity to a DID document. If starting with a handle, it is critical (mandatory) to bidirectionally verify the handle by checking that the DID document claims the handle (see atproto Handle specification). All handle resolution techniques and all atproto-blessed DID methods must be supported to ensure interoperability with all accounts.

In some client environments, it may be difficult to resolve all identity types. For example, handle resolution may involve DNS TXT queries, which are not directly supported from browser apps. Client implementations might use alternative techniques (such as DNS-over-HTTP) or could make use of a supporting web service to resolve identities.

Because authorization flows are security-critical, any caching of identity resolution should choose cache lifetimes carefully. Cache lifetimes of less than 10 minutes are recommended for auth flows specifically.

The resolved DID should be bound to the overall auth session and should be used as the primary account identifier within client app code. Handles (when verified) are acceptable to display in user interfaces, but may change over time and need to be re-verified periodically. When passing an account identifier through to the Authorization Server as part of the Authorization Request in the login_hint, it is recommended to use the exact account identifier supplied by the user (handle or DID) to ensure any sign-in flow is consistent (users might not recognize their own account DID).

At the end of the auth flow, when the client does an initial token fetch, the Authorization Server must return the account DID in the sub field of the JSON response body. If the entire auth flow started with an account identifier, it is critical for the client to verify that this DID matches the expected DID bound to the session earlier; the linkage from account to Authorization Server will already have been verified in this situation.

If the auth flow instead starts with a server (hostname or URL), the client will first attempt to fetch Resource Server metadata (and resolve to Authorization Server if found) and then attempt to fetch Authorization Server metadata. See "Authorization Server" section for server metadata fetching. If either is successful, the client will end up with an identified Authorization Server. The Authorization Request flow will proceed without a login_hint or account identifier being bound to the session, but the Authorization Server issuer will be bound to the session.

After the auth flow continues and an initial token request succeeds, the client will parse the account identifier from the sub field in the token response. At this point, the client still cannot trust that it has actually authenticated the indicated account. It is critical for the client to resolve the identity (DID document), extract the declared PDS host, confirm that the PDS (Resource Server) resolves to the Authorization Server bound to the session by fetching the Resource Server metadata, and fetch the Authorization Server metadata to confirm that the issuer field matches the Authorization Server origin (see draft-ietf-oauth-v2-1 section 7.3.1 regarding this last point).

To reiterate, it is critical for all clients - including those only interested in atproto Identity Authentication - to go through the entire Authorization flow and to verify that the account identifier (DID) in the sub field of the token response is consistent with the Authorization Server hostname/origin (issuer).

Authorization Scopes
OAuth scopes allow more granular control over the resources and actions a client is granted access to.

The special atproto scope is required for all atproto OAuth sessions. The semantics are somewhat similar to the openid scope: inclusion of it confirms that the client is using the atproto profile of OAuth and will comply with all the requirements laid out in this specification. No access to any atproto-specific PDS resources will be granted without this scope included.

Authorization Servers may support other profiles of OAuth if client does not include the atproto scope. For example, an Authorization Server might function as both an atproto PDS/entryway, and support other protocols/standards at the same time.

Use of the atproto OAuth profile, as indicated by the atproto scope, means that the Authorization Server will return the atproto account DID as an account identifier in the sub field of token requests. Authorization Servers must return atproto in scopes_supported in their metadata document, so that clients know they support the atproto OAuth profile. A client may include only the atproto scope if they only need account authentication - for example a "Login with atproto" use case. Unlike OpenID, profile metadata in atproto is generally public, so an additional authorization scope for fetching profile metadata is not needed.

The OAuth 2.0 specification does not require Authorization Servers to return the granted scopes in the token responses unless the scope that was granted is different from what the client requested. In the atproto OAuth profile, servers must always return the granted scopes in the token response. Clients should reject token responses if they don't contain a scope field, or if the scope field does not contain atproto.

The intention is to support flexible scopes based on Lexicon namespaces (NSIDs) so that clients can be given access only to the specific content and API endpoints they need access to. Until the design of that scope system is ready, the atproto profile of OAuth defines two transitional scopes which align with the permissions granted under the original "session token" auth system:

transition:generic: broad PDS account permissions, equivalent to the previous "App Password" authorization level.
write (create/update/delete) any repository record type
upload blobs (media files)
read and write any personal preferences
API endpoints and service proxying for most Lexicon endpoints, to any service provider (identified by DID)
ability to generate service auth tokens for the specific API endpoints the client has access to
no account management actions: change handle, change email, delete or deactivate account, migrate account
no access to DMs (the chat.bsky.* Lexicons), specifically
transition:chat.bsky: equivalent to adding the "DM Access" toggle for "App Passwords"
API endpoints and service proxying for the chat.bsky Lexicons specifically
ability to generate service auth tokens for the chat.bsky Lexicons
this scope depends on and does not function without the transition:generic scope
Authorization Requests
This section details standards and requirements specific to Authorization Requests.

PKCE and PAR are required for all client types and Authorization Servers. Confidential clients authenticate themselves using JWT client assertions.

Request Fields
A summary of fields relevant to authorization requests with the atproto OAuth profile:

client_id (string, required): identifies the client software. See "Clients" section above for details.
response_type (string, required): must be code
code_challenge (string, required): the PKCE challenge value. See "PKCE" section.
code_challenge_method (string, required): which code challenge method is used, for example S256. See "PKCE" section.
state (string, required): random token used to verify the authorization request against the response. See below.
redirect_uri (string, required): must match against URIs declared in client metadata and have a format consistent with the application_type declared in the client metadata. See below.
scope (string with space-separated values, required): must be a subset of the scopes declared in client metadata. Must include atproto. See "Scopes" section.
client_assertion_type (string, optional): used by confidential clients to describe the client authentication mechanism. See "Confidential Client" section.
client_assertion (string, optional): only used for confidential clients, for client authentication. See "Confidential Client" section.
login_hint (string, optional): account identifier to be used for login. See "Authorization Interface" section.
The client_secret value, used in many other OAuth profiles, should not be included.

The state parameter in client authorization requests is mandatory. Clients should use randomly-generated tokens for this parameter and not have collisions or reuse tokens across any combination of device, account, or session. Authorization Servers should reject duplicate state parameters, but are not currently required to track state values across accounts or sessions. The state parameter is effectively used to verify the issuer later, and it is important that the parameter can not be forged or guessed by an untrusted party.

For web clients, the redirect_uri is a HTTPS URL which will be redirected in the browser to return users to the application at the end of the Authorization flow. The URL may include a port number, but not if it is the default port number. The redirect_uri must match one of the URIs declared in the client metadata and the Authorization Server must verify this condition. The URL origin must match that of the client_id.

There is a special exception for the localhost development workflow to use http://127.0.0.1 or http://[::1] URLs, with matching rules described in the "Localhost Client Development" section. These clients use web URLs, but have application_type set to native in the generated client metadata.

For native clients, the redirect_uri may use a custom URI scheme to have the operating system redirect the user back to the app, instead of a web browser. Native clients are also allowed to use an HTTPS URL. Any custom scheme must match the client_id hostname in reverse-domain order. The URI scheme must be followed by a single colon (:) then a single forward slash (/) and then a URI path component. For example, an app with client_id https://app.example.com/client-metadata.json could have a redirect_uri of com.example.app:/callback.

Native clients are also allowed to use an HTTPS URL. In this case, the URL origin must be the same as the client_id. One example use-case is "Apple Universal Links".

Clients may include additional optional authorization request parameters - and servers may process them - but they are not required to. Refer to other OAuth standards and the IANA OAuth parameter registry.

Proof Key for Code Exchange (PKCE)
PKCE is mandatory for all Authorization Requests. Clients must generate new, unique, random challenges for every authorization request. Authorization Servers must prevent reuse of code_challenge values across sessions (at least within some reasonable time frame, such as a 24 hour period).

The S256 challenge method must be supported by all clients and Authorization Servers; see RFC 7636 for details. The plain method is not allowed. Additional methods may in theory be supported if both client and server support them.

Authorization Servers should reject reuse of a code value, and revoke any outstanding sessions and tokens associated with the earlier use of the code value.

Pushed Authorization Requests (PAR)
Authorization Servers must support PAR and clients of all types must use PAR for Authorization Requests.

Authorization Servers must set require_pushed_authorization_requests to true in their server metadata document and include a valid URL in pushed_authorization_request_endpoint. See RFC 9207 for requirements on this URL.

Clients make an HTTPS POST request to the pushed_authorization_request_endpoint URL, with the request parameters in the form-encoded request body. They receive a request_uri (not to be confused with redirect_uri) in the JSON response object. When they redirect the user to the authorization endpoint (authorization_endpoint), they omit most of the request parameters they already sent and include this redirect_uri along with client_id as query parameters instead.

PAR is a relatively new and less-supported standard, and the requirement to use PAR may be relaxed if it is found to be too onerous a requirement for client implementations. In that case, Authorization Servers would be required to support both PAR and non-PAR requests with PAR being optional for clients.

Confidential Client Authentication
Confidential clients authenticate themselves during the Authorization Request using a JWT client assertion. Authorization Servers may grant confidential clients longer token/session lifetimes. See "Tokens" section for more context.

The client assertion type to use is urn:ietf:params:oauth:client-assertion-type:jwt-bearer, as described in "JSON Web Token (JWT) Profile for OAuth 2.0 Client Authentication and Authorization Grants" (RFC 7523). Clients and Authorization Servers currently must support the ES256 cryptographic system. The set of recommended systems/algorithms is expected to evolve over time.

Additional requirements:

confidential clients must publish one or more client authentication keys (public key) in the client metadata. This can be either direct JWK format as JSON in the jwks field, or as a separate JWKS JSON object on the web linked by a jwks_uri URL. A jwks_uri URL must be a valid fully qualified URL with https:// scheme.
confidential clients should periodically rotate client keys, adding new keys to the JWKS set and using then for new sessions, then removing old keys once they are no longer associated with any active auth sessions
confidential clients must include token_endpoint_auth_method as private_key_jwt in their client metadata document
confidential clients are expected to immediately remove client authentication keys from their client metadata if the key has been leaked or compromised
Authorization Servers must bind active auth sessions for confidential clients to the client authentication key used at the start of the session. The server should revoke the session and reject further token refreshes if the client authentication key becomes absent from the client metadata. This means the Authorization Server is expected to periodically re-fetch client metadata.
Tokens and Session Lifetime
Access tokens are used to authorize client requests to the account's PDS ("Resource Server"). From the standpoint of the client they are opaque, but they are often signed JWTs including an expiration time. Depending on the PDS implementation, it may or may not be possible to revoke individual access tokens in the event of a compromise, so they must be restricted to a relatively short lifetime.

Refresh tokens are used to request new tokens (of both types) from the Authorization Server (PDS or entryway). They are also opaque from the standpoint of clients. Auth sessions can be revoked - invalidating the refresh tokens - so they may have a longer lifetime. In the atproto OAuth profile, refresh tokens are generally single-use, with the "new" refresh token replacing that used in the token request. This means client implementations may need locking primitives to prevent concurrent token refresh requests.

To request refresh tokens, the client must declare refresh_token as a grant type in their client metadata.

Tokens are always bound to a unique session DPoP key. Tokens must not be shared or reused across client devices. They must also be uniquely bound to the client software (client_id). The overall session ends when the access and refresh tokens can no longer be used.

The specific lifetime of sessions, access tokens, and refresh tokens is up to the Authorization Server implementation and may depend on security assessments of client type and reputation.

Some guidelines and requirements:

access token lifetimes should be less than 30 minutes in all situations. If the server cannot revoke individual access tokens then the maximum is 15 minutes, and 5 minutes is recommended.
for "untrusted" public clients, overall session lifetime should be limited to 7 days, and the lifetime of individual refresh tokens should be limited to 24 hours
for confidential clients, the overall session lifetime may be unlimited. Individual refresh tokens should have a lifetime limited to 180 days
confidential clients must use the same client authentication key and assertion method for refresh token requests that they did for the initial authentication request
Demonstrating Proof of Possession (DPoP)
The atproto OAuth profile mandates use of DPoP for all client types when making auth token requests to the Authorization Server and when making authorized requests to the Resource Server. See RFC 9449 for details.

Clients must initiate DPoP in the initial authorization request (PAR).

Server-provided DPoP nonces are mandatory. The Resource Server and Authorization Server may share nonces (especially if they are the same server) or they may have separate nonces. Clients should track the DPoP nonce per account session and per server. Servers must rotate nonces periodically, with a maximum lifetime of 5 minutes. Servers may use the same nonce across all client sessions and across multiple requests at any point in time. Servers should accept recently-stale (old) nonces to make rotation smoother for clients with multiple concurrent request in-flight. Clients should be resilient to unexpected nonce updates in the form of HTTP 400 errors and should retry those failed requests. Clients must reject responses missing a DPoP-Nonce header (case insensitive), if the request included DPoP.

Clients must generate and sign a unique DPoP token (JWT) for every request. Each DPoP request JWT must have a unique (randomly generated) jti nonce. Servers should prevent token replays by tracking jti nonces and rejecting re-use. They can restrict their client-generated jti nonce history to the server-generated DPoP nonce so that they do not need to track an endlessly growing set of nonces.

The ES256 (NIST "P-256") cryptographic algorithm must be supported by all clients and servers for DPoP JWT signing. The set of algorithms recommended for use is expected to evolve over time. Clients and Servers may implement additional algorithms and declare them in metadata documents to facilitate cryptographic evolution and negotiation.

Authorization Servers
To enable browser apps, Authorization Servers must support HTTP CORS requests/headers on relevant endpoints, including server metadata, auth requests (PAR), and token requests.

Server Metadata
Both Resource Servers (PDS instances) and Authorization Servers (PDS or entryway) need to publish metadata files at well-known HTTPS endpoints.

Resource Server (PDS) metadata must comply with the "OAuth 2.0 Protected Resource Metadata" (draft-ietf-oauth-resource-metadata) draft specification. A summary of requirements:

the URL path is /.well-known/oauth-protected-resource
response must be an HTTP 200 (not 2xx or redirect), and must be a valid JSON object with content type application/json
must contain an authorization_servers array of strings, with a single element, which is a fully-qualified URL
The Authorization Server URL may be the same origin as the Resource Server (PDS), or might point to a separate server (e.g. entryway). The URL must be a simple origin URL: https scheme, no credentials (user:password), no path, no query string or fragment. A port number is allowed, but a default port (443 for HTTPS) must not be included.

The Authorization Server also publishes metadata, complying with the "OAuth 2.0 Authorization Server Metadata" (RFC 8414) standard. A summary of requirements:

the URL path is /.well-known/oauth-authorization-server
response must be an HTTP 200 (not 2xx or redirect), and must be a valid JSON object with content type application/json
issuer (string, required): the "origin" URL of the Authorization Server. Must be a valid URL, with https scheme. A port number is allowed (if that matches the origin), but the default port (443 for HTTPS) must not be specified. There must be no path segments. Must match the origin of the URL used to fetch the metadata document itself.
authorization_endpoint (string, required): endpoint URL for authorization redirects
token_endpoint (string, required): endpoint URL for token requests
response_types_supported (array of strings, required): must include code
grant_types_supported (array of strings, required): must include authorization_code and refresh_token (refresh tokens must be supported)
code_challenge_methods_supported (array of strings, required): must include S256 (see "PKCE" section)
token_endpoint_auth_methods_supported (array of strings, required): must include both none (public clients) and private_key_jwt (confidential clients)
token_endpoint_auth_signing_alg_values_supported (array of strings, required): must not include none. Must include ES256 for now. Cryptographic algorithm suites are expected to evolve over time.
scopes_supported (space-separated string, required): must include atproto. If supporting the transitional grants, they should be included here as well. See "Scopes" section.
authorization_response_iss_parameter_supported (boolean): must be true
require_pushed_authorization_requests (boolean): must be true. See "PAR" section.
pushed_authorization_request_endpoint (string, required): must be the PAR endpoint URL. See "PAR" section.
dpop_signing_alg_values_supported (array of strings, required): currently must include ES256. See "DPoP" section.
require_request_uri_registration (boolean, optional): default is true; does not need to be set explicitly, but must not be false
client_id_metadata_document_supported (boolean, required): must be true. See "Client ID Metadata" section.
The issuer ("origin") is the overall identifier for the Authorization Server.

Authorization Interface
The Authorization Server (PDS/entryway) must implement a web interface for users to authenticate with the server, approve (or reject) Authorization Requests from clients, and manage active sessions. This is called the "Authorization Interface".

Server implementations can chose their own technology for user authentication and account recovery: secure cookies, email, various two-factor authentication, passkeys, external identity providers (including upstream OpenID/OIDC), etc. Servers may also support multiple concurrent auth sessions with users.

When a client redirects to the Authorization Server’s authorization URL (the declared authorization_endpoint), the server first needs to authenticate the user. If there is no active auth session, the user may be prompted to log in. If a login_hint was provided in the Authorization Request, that can be used to pre-populate the login form. If there are multiple active auth sessions, the user could be prompted to select one from a list, or the login_hint could be used to auto-select. If there is a single active session, the interface can move to the approval view, possibly with the option to login as a different account. If a login_hint was supplied, the Authorization Server should only allow the user to authenticate with that account. Otherwise the overall authorization flow will fail when the client verifies the account identity (sub field).

The authorization approval prompt should identify the client app and describe the scope of authorization that has been requested.

The amount of client metadata that should be displayed may depend on whether the client is "trusted" by the Authorization Server; see the "Client" and "Security Concerns" sections. The full client_id URL should be displayed by default.

See the "Scopes" section for a description of scope options.

If a client is a confidential client and the user has already approved the same scopes for the same client in the past, the Authorization Server may allow "silent sign-in" by auto-approving the request. Authorization Servers can set their own policies for this flow: it may require explicit user configuration, or the client may be required to be "trusted".

Authorization Servers should separately implement a web interface which allows authenticated users to view active OAuth sessions and delete them.

Summary of Authorization Flow
This is a high-level description of what an atproto OAuth authorization flow looks like. It assumes the user already has an atproto account.

The client starts by asking for the user’s account identifier (handle or DID), or for a PDS/entryway hostname. See "Identity Authentication" section for details.

For an account identifier, the client resolves the identity to a DID document, extracts the declared PDS URL, then fetches the Resource Server and Authorization Server locations. If starting with a server hostname, the client resolves that hostname to an Authorization Server. In either case, Authorization Server metadata is fetched and verified against requirements for atproto OAuth (see "Authorization Server" section).

The client next makes a Pushed Authorization Request via HTTP POST request. See "Authorization Request" section; some notable details include:

a randomly generated state token is required, and will be used to reference this authorization request with the subsequent response callback
PKCE is required, so a secret value is generated and stored, and a derived challenge is included in the request
scope values are requested here, and must include atproto
for confidential clients, a client_assertion is included, with type jwt-bearer, signed using the secret client authentication key
the client generates a new DPoP key for the user/device/session and uses it starting with the PAR request
if the auth flow started with an account identifier, the client should pass that starting identifier via the login_hint field
atproto uses PAR, so the request will be sent as an HTTP POST request to the Authorization Server
The Authorization Server will receive the PAR request and use the client_id URL to resolve the client metadata document. The server validates the request and client metadata, then stores information about the session, including binding a DPoP key to the session. The server returns a request_uri token to the client, including a DPoP nonce via HTTP header.

The client receives the request_uri and prepares to redirect the user. At this point, the client usually needs to persist information about the session to some type of secure storage, so it can be read back after the redirect returns. This might be a database (for a web service backend) or web platform storage like IndexedDB (for a browser app). The client then redirects the user via browser to the Authorization Server’s auth endpoint, including the request_uri as a URL parameter. In any case, clients must not store session data directly in the state request param.

The Authorization Server uses the request_uri to look up the earlier Authorization Request parameters, authenticates the user (which might include sign-in or account selection), and prompts the user with the Authorization Interface. The user might refine any granular requested scopes, then approves or rejects the request. The Authorization Server redirects the user back to the redirect_uri, which might be a web callback URL, or a native app URI (for native clients).

The client uses URL query parameters (state and iss) to look up and verify session information. Using the code query parameter, the client then makes an initial token request to the Authorization Server’s token endpoint. The client completes the PKCE flow by including the earlier value in the code_verifier field. Confidential clients need to include a client assertion JWT in the token request; see the "Confidential Client" section. The Authorization Server validates the request and returns a set of tokens, as well as a sub field indicating the account identifier (DID) for this session, and the scope that is covered by the issued access token.

At this point it is critical (mandatory) for all clients to verify that the account identified by the sub field is consistent with the Authorization Server "issuer" (present in the iss query string), either by validating against the originally-supplied account DID, or by resolving the accounts DID to confirm the PDS is consistent with the Authorization Server. See "Identity Authentication" section. The Authorization always returns the scopes approved for the session in the scopes field (even if they are the same as the request, as an atproto OAuth profile requirement), which may reflect partial authorization by the user. Clients must reject the session if the response does not include atproto in the returned scopes.

Authentication-only clients can end the flow here.

Using the access token, clients are now able to make authorized requests to the PDS ("Resource Server"). They must use DPoP for all such requests, along with the access token. Tokens (both access and refresh) will need to be periodically "refreshed" by subsequent request to the Authorization Server token endpoint. These also require DPoP. See "Tokens and Session Lifetime" section for details.

Security Considerations
There are a number of situations where HTTP URLs provided by external parties are fetched by both clients and providers (servers). Care must be taken to prevent harmful fetches due to maliciously crafted URLs, including hostnames which resolve to private or internal IP addresses. The general term for this class of security issue is Server-Side Request Forgery (SSRF). There is also a class of denial-of-service attacks involving HTTP requests to malicious servers, such as huge response bodies, TCP-level slow-loris attacks, etc. We strongly recommend using "hardened" HTTP client implementations/configurations to mitigate these attacks.

Any party can create a client and client metadata file with any contents at any time. Even the hostname in the client_id cannot be entirely trusted to represent the overall client: an untrusted user may have been able to upload the client metadata file to an arbitrary URL on the host. In particular, the client_uri, client_name, and logo_uri fields are not verified and could be used by a malicious actor to impersonate a legitimate client. It is strongly recommended for Authorization Servers to not display these fields to end users during the auth flow for unknown clients. Service operators may maintain a list of "trusted" client_id values and display the extra metadata for those apps only.

Possible Future Changes
Client metadata requests (by the authorization server) might fail for any number of reasons: transient network disruptions, the client server being down for regular maintenance, etc. It seems brittle for the Authorization Server to immediately revoke access to active client sessions in this scenario. Maybe there should be an explicit grace period?

The requirement that resource server metadata only have a single URL reference to an authorization server might be relaxed.

The details around session and token lifetimes might change with further security review.