# Introduction
# Components

## Dependencies

### Postgres

### Keycloak

The source project for the reshare-dcb configured keycloak is https://github.com/k-int/reshare-hub-keycloak

## ReShare-DCB service

The source git project for reshare-dcb-service is https://github.com/openlibraryenvironment/reshare-dcb-service/

# Installation

## Quick Start: docker-compose

The easiest way to evaluate reshare-dcb is to run the docker compose scripts provided here: https://github.com/openlibraryenvironment/reshare-dcb-devops/tree/main/docker-compose. 
Clone the repository and CD into the docker-compose directory. From there

    docker-compose up

Will launch all the components needed. The docker-compose file is also a great way to see the different environment variables that impact each component in action. Regardless
of environment, the primary way that you will configure the initial installation is through these environment variables.

## Advanced: Manual build

Reshare-DCB is targetted at JDK 17 and is built by the devs using the Temurin 17.0.6-tem distribution.

Reshare-dcb-service is a gradle build project, which has a supplied gradlew wrapper. At the time of writing, the team is running gradle 7.5.1 locally.
the Reshare-keycloak assemly is a maven project forked from https://github.com/inventage/keycloak-custom and currently being built by maven 3.8.1

The team uses sdkman to manage jdk, gradle and maven versions in development environments.

### Building the reshare keycloak service from source

Ensure JDK17 and Maven >= 3.8.1

    git clone https://github.com/k-int/reshare-hub-keycloak
    cd reshare-hub-keycloak
    ./mvnw clean install
    
    ** MAVEN WILL BUILD AND TEST **
    
    the final build artefact for stand-alone running is in server/target/keycloak/lib/quarkus-run.jar


### Building the reshare hub service from source

###

###

# Setup

Once you have a running system a number of steps need to be taken to enable the different components. This is primarily due to the need to configure keycloak. In order to 
prevent accidental default passwords, keycloak does not ship with default configuration, so some manual setup is needed here.

Visit your local keycloak installation in a browser, log in with the username from the KEYCLOAK_ADMIN environment variable, and the password from KEYCLOAK_ADMIN_PASSWORD.

Reshare-dcb is configured by default to use a keycloak realm called "reshare-hub". This is not hard-coded anywhere into the project however, and could be changed. Essentially
replacing that realm name with your local choice. These instructions will default to reshare-hub.

## Create the reshare-hub realm

- Once logged in to keycloak, click the dropdown top left (Defaulting to "Master" - the Master realm) and click the Create Realm button.
- Fill out "Realm name" with the string "reshare-hub" and click create

## Create an oauth client

- Select reshare-hub from the top left dropdown
- click clients
- click the "Create Client" button
- Fill out the client id and name as "dcb" then click next
- Select Client Authentication "On" and "Authorization" On also enable "Implicit Flow" leave other options unchecked.
- Click Save
- Visit the credentials tab of the client and take a copy of the client secret, it will be needed later on

Your oauth client is now configured. In order to enable front end javascript clients to authenticate using this endpoint, you will need to configure
the various valid redirect URLs. That configuration is deferred until later in the project.

## Realm Roles

Reshare DCB requires an administrative user with the ADMIN ROLE in order to perform administrative functions.

- Click on "Realm Roles" from the left hand menu under "Reshare-Hub"
- Click "Create Role"
- Fill out the Role Name "Admin"
- click "Save"

## Administrative user

The user we have configured at install time is the master keycloak administrator, we don't want to use that in general operations, so create a dcb admin user.

- Click on "Users" in the left hand menu
- Click on "Add User"
- Fill out the username - "admin" or your choice
- Fill out the email address and click Email verified (To skip verification)
- Fill out a first and last name just for completeness
- Click "Create"
- Once created, click on the "Credentials" tab
- Click "Set password"
- Fill out the new password, remove the temporary flag and save.

- Click on the "Role Mapping" tab
- Click "Assign Role"
- Select the "ADMIN" role and then click "Assign"

You now have an administrative user.

## Establishing a useful environment

We shold test our new user. The base URL of the endpoint for this realm (Assuming a default installation on localhost:8080 - as per the docker-compose setup) will be

    http://localhost:8080/realms/reshare-hub

At K-int we find it useful to set up a file in the users home directory called ~/.dcb.sh where we can put default configuration (This helps ensure we don't leak
credentials into git repositories). Setting up a ~/.dcb.sh with the following environment variables is helpful:

    export KEYCLOAK_BASE="http://localhost:8080/realms/reshare-hub"
    export KEYCLOAK_CERT_URL="$KEYCLOAK_BASE/protocol/openid-connect/certs"
    export KEYCLOAK_CLIENT=dcb
    export KEYCLOAK_SECRET=<<Get this value from the realm CREDENTIALS tab in keycloak>>
    export DCB_ADMIN_USER=<<YOUR_ADMIN_USER>>
    export DCB_ADMIN_PASS=<<YOUR_ADMIN_PASS>>

Once you have set up this script, you can use the helper scripts in the reshare-dcb repository: https://github.com/openlibraryenvironment/reshare-dcb-service/tree/main/scripts.
In particular, the login script should



## Test your oauth 

the login script from the reshare-dcb-service is simply

    #!/bin/bash
    source ~/.dcb.sh
    export TOKEN=`curl -s \
      -d "client_id=$KEYCLOAK_CLIENT" \
      -d "client_secret=$KEYCLOAK_SECRET" \
      -d "username=$DCB_ADMIN_USER" \
      -d "password=$DCB_ADMIN_PASS" \
      -d "grant_type=password" \
      "$KEYCLOAK_BASE/protocol/openid-connect/token" | jq -r '.access_token'`
    echo $TOKEN

But you can also just enter the command directly, for example

    curl -s -d client_id=dcb \ 
            -d client_secret=fnET5uS6i293OIwEoarQSb2N483UDXmX \
            -d username=admin \
            -d password=admin \
            -d grant_type=password \
            http://localhost:8080/realms/reshare-hub/protocol/openid-connect/token

This should return a JWT which you can cut and paste into jwt.io - Check that the returned JWT has realm_access.roles array which contains the "ADMIN" role.

# Configure reshare-dcb proper. 

Assuming all has gone well up to this point, you are ready to interact with reshare-dcb itself.

Reshare-dcb uses UUID4 for some record identifiers. In order to be able to generate UUID4s you will need a context namespace. This can be obtained with the following sequence

    export TARGET="http://localhost:8080"
    export RESHARE_ROOT_UUID=`uuidgen --sha1 -n @dns --name projectreshare.org`
    export AGENCIES_NS_UUID=`uuidgen --sha1 -n $RESHARE_ROOT_UUID --name Agency`
    export HOSTLMS_NS_UUID=`uuidgen --sha1 -n $RESHARE_ROOT_UUID --name HostLms`
    export LOCATION_NS_UUID=`uuidgen --sha1 -n $RESHARE_ROOT_UUID --name Location`

Get a TOKEN to use via the keycloak endpoint - using the login script from the scripts directory

export TOKEN=`login`

or by hand

    TOKEN=`curl -s -d client_id=dcb -d client_secret=YOUR_CLIENT_SECRET -d username=admin -d password=admin -d grant_type=password http://localhost:8080/realms/reshare-hub/protocol/openid-connect/token | jq .access_token -r`

In order to add a hostLMS which will be used in the ingest process, post to the authenticated hostlmss endpoint with the details of your hostLMS. For example

    curl -X POST $TARGET/hostlmss -H "Content-Type: application/json"  -H "Authorization: Bearer $TOKEN" -d '{ 
      "id":"'`uuidgen --sha1 -n $HOSTLMS_NS_UUID --name ARCHWAY`'", 
      "code":"MYHOSTLMS", 
      "name":"My Host LMS", 
      "lmsClientClass": "org.olf.reshare.dcb.core.interaction.sierra.SierraLmsClient", 
      "clientConfig": { 
        "base-url": "https://host.name.of.server:443",
        "key": "KEY_OF_SERVER",
        "page-size": "1000",
        "secret": "SECRET_OF_SERVER",
        "ingest": "true"
      } 
    }'





# NOTES

## reshare-dcb env variables

We're aware that there is currently some duplication in the ENV setup for reshare dcb. This is in part due to lack of flyway support for r2dbc and part due to different ways
of configuring DB connections. This will be cleaned up at some point in dev.