# SWAMP
SWAMP, or Standardised Web Account Management Protocol, is a protocol to allow an automated system to manipulate a user's account details for a web service using a well-known API.

### Why SWAMP?
A user may have dozens of web accounts, each with it's own username and password. They are being a good citizen and using a password management application to generate randomized, secure passwords. However, they also know that they should change their passwords regularly, but having so many accounts this becomes a real chore to indidually navigate to each website's account management page, generate a new password, update their password and update the stored version of their password.

This is made somewhat easier through Single Sign On based system such as OAuth, Facebook, Google etc. However, there are still many service 

Imagine if the password management application do update their password to a lovely new secure randomized password automatically on thier behalf.

This is where SWAMP comes in: By defining a standard API for each service, the password management application can log in on the user's behalf and update their password, or other account details (maybe they have changed their telephone number) automatically.

## Protocol
The set of SWAMP protocols define a well known set of services which allow an automated system to update a user's account details.

### Use Cases

#### Service Discovery
A SWAMP client can discover whether an HTTP domain implements SWAMP through a .well-known location according to [RFC5785](https://tools.ietf.org/html/rfc5785).

This is discovered for a service (domain) by an HTTP GET request to /.well-known/swamp

If the service supports SWAMP then this will return an HTTP 200 status with a JSON payload:

	{
		"version": "1.0",
		"url": "/.swamp"
	}
	
This gives the url of the SWAMP discovery service.

##### SWAMP Endpoint Discovery
The client then performs an HTTP GET to discover the capabilities of the SWAMP service, passing in an authenticated URL (one which is protected by the account which we wish to manage). Note that this is not necessarily the url for the login form or homepage of the service. It could be any url which would normally trigger authentication.

The client sends a GET request, giving the :

	GET /.swamp?url=/service1

The server responds with a JSON response:

	{
		"url" : "/.swamp/service1",
		"capabilities" : {
			"updatePassword": true,
		}
	}

This tells the client a number of pieces of information:

1. That the SWAMP service to administer this service is hosted at `/.swamp/service1` (referred to for the rest of this document as the **SWAMP endpoint**)
2. The SWAMP service is capable of changing the user's password.

#### Authentication
In order to carry out an action on behalf of the user, the client has to authenticate with the service. This is carried out one of three ways:

##### Using HTTP Basic Authentication
If the client knows the username and password to authenticate then it can authenticate using HTTP Basic authentication. This is carried out by simply providing a standard `WWW-Authenticate` header when sending the SWAMP request.

##### Using a Login Form
A password manager application may not know the username and password per se, but may only have a map of form fields to values for a login form. In order to authenticate with this information, the client sends an "authForm" payload with the JSON SWAMP request which should be a map of fieldnames to field values:

	{
		"action" : "...",
		"authForm" : {
			"user": "<username>",
			"passwordfield": "<password>",
			"someotherfield": "some inconsequential field"
		}
	}
	
Note: If the service provides multiple login forms with different field names, the server MUST either:

1. Allow either set of fields to be provided in the "authForm" object
2. Provide separate SWAMP endpoints for each of the forms, depending on which one the user would be redirected to for the discovered URL in the SWAMP Endpoint Discovery.

##### Using an Existing Session Cookie
If the client already has an authenticated session with the service (e.g. it is a web browser and the user has already logged into the service) then the client MAY provide an existing session cookie as an HTTP `Cookie` header when carrying out the SWAMP action.

##### Response - Not Authorized
If the authentication was not successful then ther server MUST return an HTTP status of 401 (Not Authorized). The server SHOULD also return a json object:

	{
		"status" : "error",
		"reason" : "auth",
		"reasonText" : "The supplied credentials could not be authorized"
	}
	
> Note: If the username/password combination could not be authenticated, the server SHOULD NOT give an indication as to whether it was specifically the username or password is correct.

#### Changing a Password
The `updatePassword` action is used to change a user's current password and is carried out by a POST to the SWAMP endpoint.

##### Request
The client POSTS the following payload:

	{
		"action" : "updatePassword",
		"authForm" : {
			... (See Authentication)
		},
		"newPassword" : "<new password>"
	}

##### Response - Password changed OK
If the action was successful and the password has been changed the server MUST respond with a status of "success".

	{
		"status": "success"
	}

##### Response - Password does not meet policy requirements
A service might have a policy to enforce certain password standards. If the password does not meet the password policy of the service then the server MUST return a `"policy"` error, and SHOULD provide a textual description of the reason the password did not comply with policy.

	{
		"status": "error",
		"reason": "policy",
		"reasonText": "The password should be at least 12 characters long"
	}

## Security Considerations
* The service SHOULD only be operated over HTTPS. If a step of the process redirects to an insecure connection then the client SHOULD terminate the process and SHOULD NOT send any credentials over an insecure link.
* The client SHOULD reject connections to a SWAMP service if the certificate is invalid
* The client might wish to allow the user to override this and prompt the user whether to connect to a system with an invalid certificates (e.g. self signed). This SHOULD be on a case by case basis and SHOULD NOT be a global setting.