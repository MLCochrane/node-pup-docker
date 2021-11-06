# NodePup
## What is this?
NodePup is a simple docker image for supporting [puppeteer](https://github.com/puppeteer/puppeteer) - an npm package, _"...which provides a high-level API to control Chrome or Chromium..."_ The base image DOES NOT include puppeteer by default, but includes NodeJS and the required packages for successfully running headless chrome.

## How to use this?
This image can be used as a base for your own images where you want to use puppeteer. Here is an example of a custom Dockerfile that would be used for e2e tests with [jest](https://jestjs.io/). For additional information on running jest and puppeteer together, please refer to the jest docs [here](https://jestjs.io/docs/puppeteer).

```docker
# Base Image
FROM mlcochrane/node-pup:1.0

WORKDIR /usr/src/app
COPY package*.json wait-it-out.sh jest.config.js ./
RUN CI=true npm install
COPY setup ./setup
COPY __tests__ ./__tests__

# Sets the env to our arg in compose so we can use it in CMD at run
ARG FRONTEND_URL
ENV FRONTEND_URL=${FRONTEND_URL}

CMD bash ./wait-it-out.sh -t "${FRONTEND_URL}" -c "npm run test"
```

This could be used alongside something such as [docker compose](https://docs.docker.com/compose/) to spin up your project for testing.

Example docker-compose.yml creating a MongoDB container, a backend, a client, and our puppeteer container.

```yaml
version: '3.7'

services:
  mongo:
    command: [--auth]
    container_name: 'mongo-container'
    restart: always
    environment:
      ...
    build:
      context: ./mongo-init

  node:
    container_name: 'server'
    depends_on:
      - mongo
    build:
      context: ./server

  web:
    container_name: 'client'
    depends_on:
      - node
    build:
      context: ./client

  puppeteer:
    container_name: 'puppeteer'
    depends_on:
      - web
    build:
      context: ./e2e
      dockerfile: Dockerfile.e2e
      args:
        FRONTEND_URL: ${FRONTEND_URL}
```

## Tips
1. While docker compose provides the `depends_on` option to imply container dependency order, it DOES NOT mean the services within are "ready." This can result it what seems like issues in services being inaccessible or tests timing out, when simply your web server hadn't started yet. To read up more on how to solve this, check the docker documention on (startup order)[https://docs.docker.com/compose/startup-order/].
2. If you're not familiar with docker and its networking you may find you're having trouble connecting to your various services as you may have previously through a more common `localhost:PORT_NUMBER` URI. Containers added to a user-defined bridge network (one is created by default with docker compose) can connect through their container names as the network provides automatic DNS resolution. In our docker comopse example above if your web server was previously available on `localhost:3000` from your server or puppeteer, it would now be available _within_ the containers at `web:3000`. To learn more about docker networking you can visit the [documention](https://docs.docker.com/network/).