CAS Overlay Template with Simple OpenID Connect POC
===================================================
The orginal README.md can be found here [README_orginal.md](./README_orginal.md)

This poc should be be and running with following small steps.
* git clone this project
* add /src/main/resources/myKeystores/thekeystore to your java cacert
    * go to ```<path-java-installation>/jre/lib/security```
    * ```keytool -import -alias oauthtestclient -keystore cacerts -file thekeystore```
    * password is 'changeit' without '
* ```./build.sh run```

Testing CAS Overlay Template with Simple OpenID Connect POC
===========================================================
* ```./build.sh run```
* https://localhost:8443/cas/oidc/authorize?response_type=code&client_id=client-test-openid&redirect_uri=https://www.google.be&scope=openid
    * login with 'tom'-'mot'
    * --> will result in https://www.google.be/?code=OC-2-FfgQpbkkiv7mXAHN2TfCa1hfv7oEVedEb46
* https://localhost:8443/cas/oidc/authorize?response_type=token&client_id=client-test-openid&redirect_uri=https://www.google.be&scope=openid
    * login with 'tom'-'mot'
    * --> will return https://www.google.be/#access_token=AT-1-Jpmtmjeip1HXjgNO2iXGhATMRdWGlcNod3H&token_type=bearer&expires_in=7200
    * copy this token(```access_token=AT-1-Jpmtmjeip1HXjgNO2iXGhATMRdWGlcNod3H```) in following url https://localhost:8443/cas/oidc/profile?access_token=<token_from_result_url_above>
        * -> will return something like:
            ```
            {
                "sub" : "tom",
                "auth_time" : 1482760969
            }
            ```
* for more possibilities see: https://apereo.github.io/cas/5.0.x/installation/OIDC-Authentication.html

History, what did I change in comparison with https://github.com/apereo/cas-overlay-template
=============================================================================================
* git clone https://github.com/apereo/cas-overlay-template.git cas-overlay-template-5.0-openid-connect
* cd cas-overlay-template-5.0-openid-connect/
    * ```./build.sh help```
    * ```./build.sh package```
* create Keystore
    * ```keytool -keystore thekeystore -genkey -alias oauthtestclient -keyalg RSA```
    * create the folder for example ```etc/cas/services``` (or put directly for testing in the project in the resources folder) and put the keystore in it
* Copying application.properties ( this can also be doen by overriding the properties with cas.properties, for more info see cas docs.)
    * Copy ```target/cas/WEB-INF/classes/application.properties``` into ```src/main/resources/application.properties```
    * add 
        ```
        cas.server.name=https://localhost:8443
        cas.server.prefix=https://localhost:8443/cas
        ```
    * change ```server.ssl.key-store=file:<path_keyStore_created_above>```
        --> it can't find thekeystore if it is copied to/etc/cas on windows
        (think about replacing casuser:Mellon at the end of the file, by your own user tom - mot)
* Test basic set up
    * ```./build.sh run```
    * => Test if you can login with tom - mot
* Making sure localhost self-signed cert authorize callback works:
    * When working locally with self-signed cert make sure it cert is present in your java keystore cacert (same as cert created above)
    * go to ```<path-java-installation>/jre1.8.0_92/lib/security```
    * ```keytool -import -alias oauthtestclient -keystore cacerts -file localhost.crt```
        * pass changeit
    * => This is needed later on when cas is calling itself for auth2.0 authorize callback
* Adding openId connect :(follow https://apereo.github.io/cas/5.0.x/installation/OIDC-Authentication.html)
    * Add following dep to pom.xml:
        ```xml
        <dependency>
            <groupId>org.apereo.cas</groupId>
            <artifactId>cas-server-support-oidc</artifactId>
            <version>${cas.version}</version>
            <scope>runtime</scope>
        </dependency>
        ```
    ~~Add following keys to application.properties
        ```
        cas.authn.oidc.issuer=https://localhost:8080/cas/oidc 
        cas.authn.oidc.skew=5
        cas.authn.oidc.jwksFile=file:<path to jwks file>  (see below)
        ```~~ (not needed)
* Adding Json service registry to be able to use services as json
    * Add following dep to pom.xml:
        ```xml
        <dependency>
            <groupId>org.apereo.cas</groupId>
            <artifactId>cas-server-support-json-service-registry</artifactId>
            <version>${cas.version}</version>
        </dependency>
        ```
    * set the correct path to your service folder in cas.serviceRegistry.config.location: application.properties
    * create the folder for example ```etc/cas/services``` (or put directly for testing in the project in the resources folder)
        * When put in resources the logging will mention ```2016-12-27 09:53:22,534 WARN [org.apereo.cas.config.JsonServiceRegistryConfiguration] - <The location of service definitions class path resource [myServices] is on the classpath. It is recommended that the location of service definitions be externalized to allow for easier modifications and better sharing of the configuration.>```
    * create a new 'service' in that folder googleBe-10025.json (<name>-<id>.json)
        * content of file is:
            ```json
            {
                "@class" : "org.apereo.cas.services.OidcRegisteredService",
                "clientId": "client-test-openid",
                "clientSecret": "secret",
                "serviceId" : "https://www.google.be",
                "signIdToken": false,
                "name": "OIDC",
                "id": 10025,
                "evaluationOrder": 103
            }
            ```
~~create jwks keystore on https://mkjwk.org/ with keyId "cas" leave 'key use' empty
    * create a file with following content 
        ```json
        {
            "alg": "RS256",
            "d": "JUqy4ioHcHnmfcjx4SRg5Zq6YH8ZQk9ZuwcPO2zAZW9AT9Ik30XWUz9H2UDD-YYGe8n6HNuZ78RzBzhFrW6zaLqRWHpOMFqAd_uREY8ldyqFjR0edaqd-9VC0JhRU7eiFDoLimEuxulCHeCotuOaVUkzv9DFVqMtZeDsGg_ltK3QOZCvBocaipfNdoJtMQ8omUAx-cGoZzD3e8EPVhRS2BXYin5-dQMg66Pi2OtFusvCr3UTvpcksfqJPRaiw7XAUCOgseJTcYSU7DMdyZ1x0bKbAGfXMleVc1NWxvgo-Uj0nTwwnAhEuGJ_OQ75ePL5j6qEJSIggMk5kyQQwa0AAQ",
            "e": "AQAB",
            "n": "puqjPsiIsseCP_s9DjOUDcVS001GWFbuUBceSmlR98-0c_8aWwiqPiGNqg9EeOJP9-U_ckUlOtKgJa7j4XHV0Zjhhrmn_1AOT3KbsmABOrX8SXqx5TvBB9rFQFxZTwJfz0HFQuf_TWJz3lbn1EuxIMG9gkO30N9OPd0i_DBAa8TkrmeN-mOBUS5ZUDejJ2nrmrpn80JBazDdj4TastKjbodbgEcShgsk0GK_kd3pRCM-oU0hfr5uyfT3V0ENAFtPorteYTK-bVTtrRYKe6B7bBdcsyjCS0s9QZCYXpzbNWBfJdRD2tdKLgPoq4WtaGcZIj6tX1up1tu767XnsBOg3Q",
            "kty": "RSA",
            "kid": "cas"
        }
        ```~~ (not needed)