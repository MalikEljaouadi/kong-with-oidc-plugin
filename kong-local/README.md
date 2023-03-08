# KONG with oidc plugin for local setup:

Runing the provided docker-compose in this directory will provide:

- KONG instance (with OIDC plugin)
- KONGA Dashboard to overview KONG
- 1st Postgres instance used by KONG
- Keycloack instance
- 2nd Postgres instance used by Keycloak

The mentioned resources above, would set the complete infrastructure to achieve the communication betwenn KONG and Keycloak