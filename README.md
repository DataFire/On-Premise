# DataFire.io On-Prem and White-Label
This is a sample repository for running DataFire.io on your own servers.
You'll need to [contact the DataFire team](https://app.datafire.io/contact) for an access key.

## Setup
```bash
sudo apt-get update
sudo apt-get install -y curl git python-pip python-dev build-essential zip gzip
pip install awscli --upgrade --user

# Use the keys provided by the DataFire team to log into docker
sudo `AWS_ACCESS_KEY_ID=YOUR_KEY AWS_SECRET_ACCESS_KEY=YOUR_SECRET aws ecr get-login --no-include-email --region us-west-2`
```

## Run the Backend

### Start a MongoDB instance
Users and project data will be stored in MongoDB.
You'll need to create a directory to store the database (here `~/datafire-mongodb`).

```bash
mkdir ~/datafire-mongodb
sudo docker run --name datafire-mongo -v ~/datafire-mongodb:/data/db -d mongo
```

### Start a Git server
Git is used to store the code for each project.
You'll need to create a directory to store projects (here `~/datafire-git`).

```bash
mkdir ~/datafire-git
git clone --bare https://github.com/DataFire-repos/hello-world ~/datafire-git/_start.git
sudo docker run -d  \
  --name datafire-git \
  -v ~/datafire-git:/var/lib/git \
  -p 3002:80 \
  cirocosta/gitserver-http

# Touch the _start repo to initialize /refs/heads/master
git clone http://localhost:3002/_start.git && cd _start
echo " " >> README.md && git commit -a -m "touch readme"
git push && cd .. && rm -rf _start
```

### Start Docker (DinD)
Docker is used to run projects (both in `dev` and `prod`). Alternatively, you can set
AWS credentials in `./backend/DataFire-accounts.yml` to use AWS ECS and Lambda.

```bash
sudo docker run --privileged --name datafire-docker -p 32768-33000:32768-33000 -d docker:17-dind
sudo docker pull 205639412702.dkr.ecr.us-west-2.amazonaws.com/datafire:latest
sudo docker save 205639412702.dkr.ecr.us-west-2.amazonaws.com/datafire | gzip > ./datafire-image.tar.gz
sudo docker cp ./datafire-image.tar.gz datafire-docker:/home/dockremap/datafire-image.tar.gz
sudo docker exec --privileged -it datafire-docker sh -c 'docker load < /home/dockremap/datafire-image.tar.gz'
```

### Update Backend Settings

Now we need to tell the DataFire backend where each of these services live.

Use `docker inspect` to find the IP address for each container:
```bash
sudo docker inspect datafire-mongo  | grep IPAddress
```

Add your MongoDB location to `./backend/DataFire-accounts.yml`. For example:


```yaml
mongodb:
    url: 'mongodb://172.17.0.2:27017'
    integration: mongodb
```

```bash
sudo docker inspect datafire-docker | grep IPAddress
sudo docker inspect datafire-git | grep IPAddress
```

Add your Docker location to `./backend/settings.js`. You should also set
your machine's public or intranet IP address as `api_host` and `web_host`. For example:

```js
modle.exports = {
  web_host: 'http://datafire.acme-co.com',
  api_host: 'http://datafire.acme-co.com:3001',
  docker_host: 'http://172.17.0.4:2375',
  git_host: 'http://172.17.0.5:3002',
}
```

### Repository Access
By default, project code is only accessible to the project's creator.
However, this means the backend **must be served over SSL**, or `git clone`
will not work, and you will not be able to deploy your projects.

If you don't have SSL set up (i.e. `api_host` doesn't start with `https://`),
you need to change `allow_public_repo_access` to `true`
in `./backend/settings.js`. Credentials will still be kept private - only the project's
code will be available publicly.

### Start the Backend
The backend will handle all API requests. Here we start it on port `3001`.

```bash
sudo docker build ./backend -t my-datafire-backend
sudo docker run -d \
  --name datafire-backend \
  -p 3001:8080 \
  -v ~/datafire-git:/var/lib/git \
  my-datafire-backend forever server.js
```

## Run the Website

### Update website settings

We need to tell the website where the API is hosted, and where deployments will live (i.e. the DinD IP address).
We also need to set `git_host` to be the same as `api_host`.

Unless you're just visiting the website on `localhost`, this should be the public or intranet IP of your server.

Edit `./web/settings.ts`
```ts
export const settings:any = {
  whitelabel: true,
  web_host: 'http://localhost',        # The URL where the website will be hosted
  api_host: 'http://localhost:3001',   # The address of the API server above
  git_host: 'http://localhost:3001',   # Usually, the same address as the API server (which proxies the git server)
  deployment_host: 'http://localhost', # The address where DIND can be reached (usually the same address as the API server, with no port)
}
```

```bash
sudo docker build ./web -t my-datafire-web
sudo docker run --name datafire-web -it -p 3000:8080 -d my-datafire-web npm run serve:prod
```

Visit `http://localhost:3000` to see the website.

### Customization

### Styles
You can set the Bootstrap theme by editing ./bootstrap.scss. You can use a [GUI editor](http://bbrennan.info/strapping/) to generate the SASS file.

Be sure to also set the variable `$brand-secondary` in bootstrap.scss.

Additional styles can be added to this file as well.

### Assets
Everything in the `./web/assets` folder will be made available at `http://localhost/assets`

### Settings
Edit `./web/settings.ts` to change your deployment's configuration

#### Options
* whitelabel: must be set to true
* logo: a URL for your company's logo
* web_host: where the website is hosted, e.g. 'https://app.datafire.io'
* api_host: where the DataFire.io backend is hosted, e.g. 'https://api.datafire.io'
* callback_url: URL for OAuth callbacks, e.g. 'https://api.datafire.io/oauth/provider/callback'
* refresh_url: URL for getting OAuth refresh tokens, e.g. 'https://api.datafire.io/oauth/provider/refresh'
* integrations: A whitelist of integrations available in the UI
* client_ids: A list of client IDs for OAuth-enabled integrations

## OAuth Applications

This is a partial list of OAuth providers that are supported. Add your `client_id` for each app
to `./web/settings.ts`, and both `client_id` and `client_secret` to `./backend/DataFire-accounts.yml`

* azure
* authentiq
* bitbucket
* box
* bufferapp
* ebay
* flat
* furkot
* github
* google
* instagram
* lyft
* medium
* motaword
* netatmo
* reverb
* runscope
* slack
* spotify
* squareup
* stackexchange
* facebook
* fitbit
* producthunt
* heroku
* linkedin
* reddit

