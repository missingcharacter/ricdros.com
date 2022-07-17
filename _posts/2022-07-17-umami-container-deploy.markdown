---
title:  'Deploy Umami website analytics on Heroku via container'
toc: true
date:   2022-07-17 00:00:00 +0000
---
## Requirements

- [A Heroku Account](https://heroku.com/)
- [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli)
- A docker runtime
  - [Docker Desktop](https://www.docker.com/products/docker-desktop/)
  - [docker cli](https://github.com/docker/cli) + [colima](https://github.com/abiosoft/colima)

## Initial App Setup

- Login to your [Heroku Account](https://heroku.com/)
- From the dashboard page click **New > Create new app**
- Choose an **App name** and then click **Create app**

### Database

- Navigate to the **Resources** tab and click on the **Find more add-ons** button
- Search for **Heroku Postgres** and follow its instructions to install the
  add-on
- The add-on will set the `DATABASE_URL` automatically; you should not have to
  manually set it
- You will need to set up the database tables by following the **Create database
  tables** section of the [Install](https://umami.is/docs/install) docs
- You can find temporary connection details by following the **Resources >
  Heroku Postgres > Settings > Database Credentials** path

## Deploy

- Under the **Settings > Config Vars** section, set the `HASH_SALT` environment
  variable. Read the [Install](https://umami.is/docs/install) section for
  information about the `HASH_SALT` environment variable.
- login to heroku cli:

  ```shell
  $ heroku login
  heroku: Press any key to open up the browser to login or q to exit:
  Opening browser to https://cli-auth.heroku.com/auth/cli/browser/
  Logging in... done
  Logged in your@email.com
  ```

- docker login to heroku repo

  ```shell
  $ heroku container:login
  Login Succeeded
  ```

- Pull latest postgresql umami version

  ```shell
  docker pull docker.umami.is/umami-software/umami:postgresql-v1.33.3
  ```

- Tag the container image with your app-name and process-type should be `web`

  ```shell
  docker tag docker.umami.is/umami-software/umami:postgresql-v1.33.3 \
    registry.heroku.com/<App name>/web
  ```

- Push the container image to heroku

  ```shell
  docker push registry.heroku.com/<App name>/web
  ```

- Create a new release so the new image is promoted

  ```shell
  $ heroku container:release web -a <App name>
  Releasing images web to <App name>... done
  ```

- Once the release has finished, the website should be live. Follow the
  **Open app** button at the top of the dashboard to view it
- Follow the **Getting started** guide starting from the
  [Login](https://umami.is/docs/login) step

## Related links

- [Umami running on Heroku](https://umami.is/docs/running-on-heroku)
- [Heroku - Docker Deploys](https://devcenter.heroku.com/articles/container-registry-and-runtime)
