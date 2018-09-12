# Example OAuth2 Authorization Code Flow with Vert.x

This example is built using the OAuth2 client capabilities shipped as part of the Vert.x "Web" package, specifically the [OAuth2AuthHandler](https://vertx.io/docs/vertx-web/groovy/#_oauth2authhandler_handler). If you are using Vert.x to build your OAuth2 client application, this small example should help you get started quickly.

## Quick Start Sample Code

Copy and paste from these samples to get started building your own Vert.x OAuth 2 client.

### Configuring the Auth Provider:

This is the core configuration that you will need to specify to match your use:

    def authProvider = OAuth2Auth.create(vertx, OAuth2FlowType.AUTH_CODE, [
        site:"http://am-service.sample.svc.cluster.local:80/openam",
        clientID: "vertxClient", // replace with your client id
        clientSecret: "vertxClientSecret", // replace with your client secret
        tokenPath:"/oauth2/access_token",
        authorizationPath:"/oauth2/authorize",
        introspectionPath:"/oauth2/introspect",
        useBasicAuthorizationHeader: false
    ])

    router.route().handler(UserSessionHandler.create(authProvider))

    def oauth2Handler = OAuth2AuthHandler.create(authProvider, "http://localhost:8888")
    oauth2Handler.setupCallback(router.get("/callback"))

    // scopes we want to request during login
    oauth2Handler.addAuthority("profile")
    oauth2Handler.addAuthority("openid")


This is an example which demonstrates how you would declare a protected area within your application. You can use the tokens you have gotten back from the AS to identify the user and also make requests on their behalf to resource server endpoints:

    router.route("/protected")
        .handler(oauth2Handler)
        .handler({ routingContext ->
            // At this point we are logged in using the AM provider.
            // The access and id tokens are saved in the session.
            def user = routingContext.user()

            // If we want to use the details from the id_token to make local authorization
            // decisions, we can get them with user.idToken()
            user.setTrustJWT(true)
            def id_token = user.idToken()

            // Putting the prettily-encoded token into the context so it can be
            // shown in the template, for demo purposes
            routingContext.put("idTokenDetails", (new JsonObject(id_token)).encodePrettily())

            // We can use the access_token associated with the user to make
            // requests to any resource server endpoint which is expecting
            // tokens from AM. For example, these IDM endpoints:
            user.fetch("http://rs-service.sample.svc.cluster.local/openidm/info/login", { infoResponse ->
                if (infoResponse.failed()) {
                    routingContext.response().end("Unable to read info login")
                } else {
                    def infoDetails = infoResponse.result().jsonObject()
                    def userPath = "${infoDetails.authorization.component}/${infoDetails.authorization.id}"
                    user.fetch("http://rs-service.sample.svc.cluster.local/openidm/${userPath}", { userResponse ->
                        if (userResponse.failed()) {
                            routingContext.response().end("Unable to read user details")
                        } else {
                            routingContext.put("user", userResponse.result().jsonObject())
                            routingContext.next()
                        }
                    })
                }
            })
        })


The [app.groovy](src/app.groovy) code has the full context of these two snippets. The intent is to merely provide a working example that you can refer to when building your own Vert.x application.

## Running the sample

### Prerequisites

1. Install and run the [Platform sample](https://github.com/ForgeRock/forgeops/tree/master/samples/fr-platform)

2. Register *vertxClient* application with AM as a new OAuth2 Client

Sign in:
```
curl 'http://am-service.sample.svc.cluster.local/openam/json/realms/root/authenticate' -X POST -H 'X-OpenAM-Username:amadmin' -H 'X-OpenAM-Password:password'
```

Note *tokenId* key in the results:

>{"tokenId":"AQIC5wM...3MTYxOA..*"...

Register a new OAuth2 public client. Note that you need to assign the above tokenId value to *iPlanetDirectoryPro* cookie:

```
curl 'http://am-service.sample.svc.cluster.local/openam/json/realms/root/realm-config/agents/OAuth2Client/vertxClient' -X PUT --data '{"coreOAuth2ClientConfig":{"userpassword":"vertxClientSecret","redirectionUris":["http://localhost:8888/callback"],"scopes":["openid","profile"],"tokenEndpointAuthMethod":"client_secret_post"}}' -H 'Content-Type: application/json' -H 'Accept: application/json' -H 'Cookie: iPlanetDirectoryPro=AQIC5wM...3MTYxOA..*'
```
>{"_id":"vertxClient", . . . "_type":{"_id":"OAuth2Client","name":"OAuth2 Clients","collection":true}}

Alternatively you can add *vertxClient* manually, utilizing the platform UI: [AM Console](http://am-service.sample.svc.cluster.local/openam/console)

* Sign in with *amadmin/password*
* Navigate to *Top Level Realm* > *Applications* > *OAuth 2.0
* Add new client:
    * "Client ID": "vertxClient"
    * "Client Secret": "vertxClientSecret"
    * "Redirection URIs": ["http://localhost:8888/callback"]
    * "Scope(s)": ["openid", "profile"]
* Click Save, then go to "Advanced"
    * "Token Endpoint Authentication Method": "client_secret_post"


### Serve the application for this sample

The easiest way to execute this sample is by using Docker. This will automate the download and setup of your Vert.x execution environment.

    docker build -t basicvertxclient:latest .
    docker run -d -p 8888:8888 -p 5005:5005 basicvertxclient:latest

On Linux, you need to provide a separate flag (*--network host*) to get local host name resolution to work properly:

    docker run -d -p 8888:8888 -p 5005:5005 --network host basicvertxclient:latest

Now you can access the application with http://localhost:8888