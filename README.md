[![GitHub Release](https://img.shields.io/github/v/release/zloom/keycloak-external-claim-mapper?color=blue)](https://github.com/zloom/keycloak-external-claim-mapper/releases)
[![GitHub license](https://img.shields.io/badge/License-Apache-blue.svg)](https://github.com/zloom/keycloak-external-claim-mapper/blob/main/LICENSE)
# Keycloak External Claim Mapper
Implementation of the keycloak internal SPI protocol-mapper, that allows to fetch remote http json data and include it into user JWT.
## Limitations
Built and tested with Keycloak 25.0.0, maven 3.6.3, openjdk21, you can build and test with other versions if required.
## Installation
To build Keycloak 25.0.0 image with extension from 0.0.2 release create dockerfile with following content:
```
FROM alpine:3.20 as build

ARG VERSION=0.0.2

RUN \
  wget https://github.com/zloom/keycloak-external-claim-mapper/releases/download/${VERSION}/external.claim.mapper-${VERSION}.tar.gz;\
  mkdir -p /providers;\
  tar -C /providers -zxvf external.claim.mapper-${VERSION}.tar.gz;

FROM quay.io/keycloak/keycloak:25.0.0 as keycloak

COPY --from=build /providers /opt/keycloak/providers

ENV KEYCLOAK_ADMIN="admin"
ENV KEYCLOAK_ADMIN_PASSWORD="admin"
```
Build and run it with following command:
```
docker build -t keycloak . && docker run -p 8080:8080 keycloak start-dev
```
Login and password are `admin`. Check `http://localhost:8080/admin/master/console/#/master/providers` and find `external-claim-mapper` if extension is picked up by Keycloak, it should be there. Check detailed development setup guide [here](https://www.zloom.org/blogs/debugging-keycloak-extension?utm_source=keycloak-external-claim-mapper).
## Configuration
There are 5 fields to configure, except standard protocol mapper provider configuration.
### Remote url
This is remote url to fetch json from, you can optionally use a placeholders to insert keycloak user id, user name and email. 
- Placeholders will work as following: assume your remote urls is `http://localhost/user` so that `http://localhost/user/**uid**/**uname**/**email**` will be transformed to `http://localhost/user/063943d2-d7ed-4bca-812b-506518c38228/test/test@example.com` given that `063943d2-d7ed-4bca-812b-506518c38228` is keycloak user id, `test` is user name and `test@example.com` is the email.
### Json path
Optional json path expression to transform your remote endpoint response data.
Given that **Token Claim Name** is configured as `user_roles` and remote endpoint response is:
```
{
  "roles": {
    "values": [
      "role1",
      "role2",
      "role3"
    ]
  }
}
```
In final jwt it will look as following:
```
...
"user_roles": {
  "roles": {
    "values": [
      "role1",
      "role2",
      "role3"
    ]
  }
}
...
```
You can set **Json path** to `$.roles.values` it will result to:
```
...
"user_roles": [
  "role1",
  "role2",
  "role3"
]
...
```
It is convenient when you don't have control over remote endpoint response shape, you can test json path expressions with https://jsonpath.com
### Error propagation
A switch to propagate errors that may occurs during data fetch and transformation. Mapper will fail for result code not from 2XX or 3XX codes. This flag also will propagate error on jsonpath transformation. Error will be propagated to keycloak token endpoint that will cause 500 error on auth. Exact error should be visible in keycloak logs.
### User authentication
When this options is enabled, extension will add user session JWT token, of course this token will not contain remote claims yet. Token will be added with following header `"Authorization" : "Bearer eyJ..."`
### Request headers
Set of key value pairs which you can use to configure custom request headers, when multiple values provided with same key header value will be joined with comma as following `"X-key": "value1, value2"`. You can use `**uid**` and `**uname**` placeholders same like in remote url.
## License
This project is licensed under the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0)

## Development

Use Docker Compose to build and run changes in a local test environment

```docker compose build```

```docker compose up```

This launches a Keycloak server loaded up with the module installed, a Postgresql database and a mocked server which serves user roles.

Login to Keycloak with `admin` as the username and password and select the `dev` realm. A test user (username: `test`, password: `test`) is preconfigured and the custom mapper is loaded into the `account` client. Visit `Client Scopes -> account-dedicated -> external_claims` to change the settings.

Run the following to generate the JWT token loaded with the mocked claims

```
curl -X POST http://localhost:8080/realms/dev/protocol/openid-connect/token -H 'Content-Type: application/x-www-form-urlencoded' -d grant_type=password -d client_id=account -d username=test -d password=test
```

