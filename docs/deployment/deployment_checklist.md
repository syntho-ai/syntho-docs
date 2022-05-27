# Deployment checklist

## Frontend

- Needs to be publicly available (Ingress/Loadbalancer needed for Kubernetes)
- Maybe see if we can format the ENV variables to a little less variables
- Frontend available via only IP (and not ingress) in Helm

## Backend

- Add environment variables for database connection
- Add secret key to environment variables

## Core API

- Split database_url environment variable into appropriate variables (host, database, password, user)


## Other services