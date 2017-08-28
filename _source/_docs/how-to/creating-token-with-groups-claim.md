---
layout: docs_page
title: How to Create a Token with a Groups Claim
excerpt: How to create an ID token or access token that contains a groups claim
---

## How to Create a Token with a Groups Claim

You can add a groups claim for any app or user group into ID tokens or access tokens for API Access Management.
This process optionally uses Okta's flexible profile, which accepts any JSON-compliant content, to create a whitelist of groups
that can then easily be referenced. This is especially useful if you have a large number of groups to whitelist or otherwise
need to dynamically set group whitelists on a per-application basis.

### Before You Start

* Create an OpenID Connect client with the [Apps API](/docs/api/resources/apps.html#request-example-8). In the instruction examples, the client ID is `0oabskvc6442nkvQO0h7`.
* Create the groups that you wish to configure in the groups claim. In the instruction examples, we're configuring the `WestCoastDivision` group, and the group ID is `00gbso71miOMjxHRW0h7`.

### Create a Token with a Groups Claim

You'll assign a group whitelist to your OpenID Connect client, and configure a groups claim that references the whitelist.

1. Get the group IDs you want to whitelist: `/{yourOktaDomain}.com/api/v1/groups`
   Note that this example uses one `groupId` for simplicity's sake.

   Request Example:
    ~~~
    curl -X GET \
      -H 'Accept: application/json' \
      -H 'Authorization: SSWS ${api_token}' \
      -H 'Content-Type: application/json' \
      "http://{yourOktaDomain}.com/api/v1/groups"
    ~~~
    
    Response Example:
    ~~~json
     [
        {
                "id": "00gbso71miOMjxHRW0h7",
                "created": "2017-08-25T21:15:48.000Z",
                "lastUpdated": "2017-08-25T21:15:48.000Z",
                "lastMembershipUpdated": "2017-08-25T21:16:07.000Z",
                "objectClass": [
                    "okta:user_group"
                ],
                "type": "OKTA_GROUP",
                "profile": {
                    "name": "WestCoastDivision",
                    "description": "Employees West of the Rockies"
                },
                "_links": {
                    "logo": [
                        {
                            "name": "medium",
                            "href": "https://op1static.oktacdn.com/assets/img/logos/groups/okta-medium.d7fb831bc4e7e1a5d8bd35dfaf405d9e.png",
                            "type": "image/png"
                        },
                        {
                            "name": "large",
                            "href": "https://op1static.oktacdn.com/assets/img/logos/groups/okta-large.511fcb0de9da185b52589cb14d581c2c.png",
                            "type": "image/png"
                        }
                    ],
                    "users": {
                        "href": "https://{yourOktaDomain}.com/api/v1/groups/00gbso71miOMjxHRW0h7/users"
                    },
                    "apps": {
                        "href": "https://{yourOktaDomain}.com/api/v1/groups/00gbso71miOMjxHRW0h7/apps"
                    }
                }
            }
     ]
    ~~~
    

2. If you only have one or two groups to specify, simply add the group IDs to the Okta EL function in the next step.
   However, if you have a lot of groups to whitelist, you can put the group IDs in the client app's profile property bag: `/{yourOktaDomain}.com/api/v1/apps/:aid`.
   This example names the group whitelist `groupwhitelist`, but you can name it anything.
    
    Request Example:
    ~~~curl
    curl -X POST \
      http://{yourOktaDomain}.com/api/v1/apps/xfnIflwIn2TkbpNBs6JQ \
      -H 'accept: application/json' \
      -H 'authorization: SSWS ${api_token}' \
      -H 'cache-control: no-cache' \
      -H 'content-type: application/json' \
      -d '{
        "name": "oidc_client",
        "label": "Sample Client",
        "status": "ACTIVE",
        "signOnMode": "OPENID_CONNECT",
        "credentials": {
          "oauthClient": {
            "token_endpoint_auth_method": "client_secret_post"
          }
        },
        "settings": {
          "oauthClient": {
            "client_uri": "http://localhost:8080",
            "logo_uri": "http://developer.okta.com/assets/images/logo-new.png",
            "redirect_uris": [
              "https://example.com/oauth2/callback",
              "myapp://callback"
            ],
            "response_types": [
              "token",
              "id_token",
              "code"
            ],
            "grant_types": [
              "implicit",
              "authorization_code"
            ],
            "application_type": "native"
          }
        },
        "profile": {
            "groupwhitelist": [
                "00gbso71miOMjxHRW0h7"
                ]
        }
      }
    ~~~
    
    You can select any group or set of groups, either app groups or user groups. 

    To use this group whitelist for every client that gets this claim in a token, put the `groupId` in the first parameter of the `getFilteredGroups` function described below. 
 
3. Add a custom claim for ID token on a Custom Authorization Server with the following function: `getFilteredGroups({group ID list}, {value-to-represent-the-group-in-the-token}, {maximum number of groups to include in token. Must be less than 100.})`.
    
    ~~~curl
        curl -X POST \
          http://{yourOktaDomain}.com/api/v1/authorizationServers/ausain6z9zIedDCxB0h7/claims \
          -H 'accept: application/json' \
          -H 'authorization: SSWS ${api_token}' \
          -H 'cache-control: no-cache' \
          -H 'content-type: application/json' \
          -d '{
        	"name": "groupWhitelist",
        	"status": "ACTIVE",
        	"claimType": "RESOURCE",
        	"valueType": "EXPRESSION",
        	"value": "\"getFilteredGroups(app.profile.groupwhitelist, group.name, 40)\"",
        	}
        }'
    ~~~
          
    You can also see this value in the Okta UI for claims, under **Mapping**: `getFilteredGroups(app.profile.groupwhitelist, "group.name", 40)`.
    
    See [group function documentation](/reference/okta_expression_language/#group-functions) for more information about specifying groups with `getFilteredGroups`.

4. POST to the token endpoint `/{yourOktaDomain}.com/oauth2/:authorizationServerId/v1/token` with the user or client you want. Make sure the user is assigned to the app and to one of the groups from your whitelist.

    ~~~curl
    curl -X POST \
      http://{yourOktaDomain}.com/oauth2/ausain6z9zIedDCxB0h7/v1/token \
      -H 'accept: application/json' \
      -H 'authorization: Basic e3tjbGllbnRJZH19Ont7Y2xpZW50U2VjcmV0fX0=' \
      -H 'cache-control: no-cache' \
      -H 'content-type: application/x-www-form-urlencoded' \
      -d 'grant_type=authorization_code&redirect_uri=%7B%7BredirectUri%7D%7D&code=%7B%7BauthorizationCode%7D%7D'
      ~~~

5. Decode the JWT in the response to see that the groups are in the token. 

