# API Platform JWT authentication example

An example Symfony 6 application implementing:
  - JWT Authentication for API Platform resources
  - LDAP Authentication to get a JWT+Refresh token
  - Routes to authenticate, refresh, invalidate

A docker-compose configuration is provided that spins up the application and required services (MariaDB and OpenLDAP).

This example uses JWT tokens with a low TTL. The idea is that consumers would store the JWT tokens in memory.
A refresh token is provided as a HTTP cookie. This refresh token has a higher TTL, allowing consumers a method to
generate new JWT tokens as they expire. The refresh tokens are single use, and a new refresh token is provided
upon refresh as well.

## Start the application

First, clone this repository, install and build dependencies:

```
composer install
```

Then, spin up the app and required services:

```
docker compose up --build -d
```

You can now access the app on https://localhost/.

Services running in docker expose their ports so the local application can consume them as well.
This way, you can use console command from outside docker, etc.

## Usage

- Call an API resource and see that you need authentication:

```
❯ curl -k -X GET https://localhost/api/
{"code":401,"message":"JWT Token not found"}

❯ curl -k -X POST https://localhost/api/token/refresh
{"code":401,"message":"Missing JWT Refresh Token"}

❯ curl -k -X POST https://localhost/api/token/invalidate
{"code":400,"message":"No refresh_token found."}
```

- Authenticate using the credentials in LDAP to get a JWT token:

```
❯ curl -ik -X POST https://localhost/api/token/auth \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin@example.org","password":"admin"}'
HTTP/2 200
set-cookie: refresh_token=<refresh_token>; Max-Age=7200; path=/api/token/; secure; httponly; samesite=lax
[..]
{"token":"<jwt_token>"}
```

- Use the JWT token to get an API resource:

```
❯ curl -k -X GET https://localhost/api -H 'Authorization: Bearer <jwt_token>'
{"@context":"\/api\/contexts\/Entrypoint","@id":"\/api","@type":"Entrypoint"}
```

- Use the refresh token to get a new JWT token:

```
❯ curl -ik -X POST https://localhost/api/token/refresh -b 'refresh_token=<refresh_token>'
HTTP/2 200
set-cookie: refresh_token=<new_refresh_token>; Max-Age=7200; path=/api/token/; secure; httponly; samesite=lax
[..]
{"token":"<new_jwt_token>"}
```

- Invalidate the refresh token (eg. user logout)

```
❯ curl -ik -X POST https://localhost/api/token/invalidate -b 'refresh_token=<refresh_token>'
HTTP/2 200
set-cookie: refresh_token=deleted; Max-Age=7200; path=/api/token/; secure; httponly; samesite=lax
{"code":200,"message":"The supplied refresh_token has been invalidated."}
```
