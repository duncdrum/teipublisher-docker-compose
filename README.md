# Docker Compose Configuration

This repository contains a docker compose configuration useful to run TEI Publisher and associated services. Docker compose allows us to orchestrate and coordinate the various services, while keeping each service in its own, isolated environment. Setting up a server via docker compose is fast as everything comes preconfigured and you don't need to install all the dependencies (like Java, eXist-db, Python etc.) by hand. On the downside, it certainly introduces some overhead and may never be as fast as a server, which is properly maintained. For smaller, low-traffic projects docker is a viable and cheap alternative though.

For security reasons, it is recommended to not expose TEI Publisher and eXist-db directly, but instead protect them behind a proxy. The [docker-compose](docker-compose.yml) file therefore sets up an nginx reverse proxy.

The following services are configured by the [docker-compose](docker-compose.yml):

* publisher: main TEI Publisher application
* ner: TEI Publisher named entity recognition service
* frontend: nginx reverse proxy which forwards requests to TEI Publisher
* certbot: letsencrypt certbot required to register an SSL certificate

You only need to clone this repository to either your local machine or a server you are installing. Everything else is handled automatically by docker compose.

# Default Configuration

The configuration currently builds TEI Publisher from the master branch. Once TEI Publisher 8 has been released, you will also be able to use that version. The named entity recognition service is pulled as image from the corresponding github package repository. Again the master version is required as there has not been an official release yet. If you do not need or want the named entity recognition service, comment out the corresponding section in `docker-compose.yml`, including the `depends_on: ner` above. TEI Publisher will still work.

By default, the compose configuration will launch the proxy on port 80 of the local host, serving only http, not https. This configuration is intended for testing, not for deployment on a public facing server.

# Running

To build all services, call

```sh
docker compose build --build-arg ADMIN_PASS=my_pass
```

where `my_pass` sets the password for the eXist admin user (recommended). You can remove the `--build-arg` parameter entirely to keep an empty password.

To start, simply call

```sh
docker compose up -d
```

Afterwards you should be able to access TEI Publisher using http://localhost. Additionally eXide can be accessed via http://localhost/apps/eXide (on a production system you want to disable that).

# Deployment on a Public Server

If you would like to deploy the configuration to a public server, you must first acquire an SSL certificate to enable users to securly connect via https. The compose configuration is already prepared to make this as easy as possible.

1. Clone this repository to a folder on the server
1. Copy the nginx configuration file [conf/example.com.tmpl](conf/example.com.tmpl) to e.g. `conf/my.domain.com.conf`, where `my.domain.com` would correspond to the domain name of the webserver you are configuring the service for
2. Open the copied file in an editor and replace all occurrences of `example.com` with your domain name. *Important*: this also applies to the commented out SSL section, which you will enable later below.
3. Change the name of the **upstream** entry to a unique name (otherwise it will collide with the default config):
   ```
    upstream docker-publisher.example.com {
        server publisher:8080 fail_timeout=0;
    }
    ```

    Change the two references to the `docker-publisher` upstream server below accordingly (including the commented out SSL section):

    ```
    proxy_pass http://docker-publisher.example.com/exist/apps/tei-publisher$request_uri;
    ...
    proxy_pass http://docker-publisher.example.com/exist$request_uri;
    ```
4. Start the services to acquire SSL certificates in the next step using `docker compose up -d`
5. Run the following command to request an SSL certificate for your domain, again replacing the final `example.com` with your domain name:
   ```sh
   docker compose run --rm  certbot certonly --webroot --webroot-path /var/www/certbot/ -d example.com
   ```

   This will ask you for an email address, verify your server and store certificate files into `certbot/conf/`.

6. In the nginx configuration file, uncomment the SSL section by removing the leading `#`
7. Stop and restart the services:
   ```sh
   docker compose restart
   ```

# Customize the Configuration

The default configuration exposes the TEI Publisher application itself. Instead you may want to build and deploy a custom application generated by TEI Publisher. The necessary steps are:

1. create a Dockerfile for your custom application
2. fork this repository and modify it to build your application instead of TEI Publisher
3. clone your customized repository to a server of your choice

## 1. Create a Dockerfile

We would suggest to copy TEI Publisher's main [Dockerfile](https://github.com/eeditiones/tei-publisher-app/blob/bdffd983b84297f296145e16687a59841aef5161/Dockerfile#L55) to the root of your custom app repository: 
remove the sections referring to the Shakespeare and Van Gogh demo apps, and replace the bits referring to TEI Publisher with your own custom app. By going through the file it should be easy to see. The relevant lines you should have are:  

```
ARG MY_EDITION_VERSION=1.0.6
...
# Build my-edition
RUN  git clone https://github.com/my-github-user/my-edition.git \
    && cd my-edition \
    && echo Checking out ${MY_EDITION_VERSION} \
    && git checkout ${MY_EDITION_VERSION} \
    && ant
...
COPY --from=tei /tmp/my-edition/build/*.xar /exist/autodeploy/
```

Push the final Dockerfile to your git repo.

## 2. Fork this repo and customize it

Fork `tei-publisher-docker-compose` to your own git account and clone it to apply some modifications:

1. edit `docker-compose.yml` and replace `services.publisher.build.context` to point to the repository in which your custom application lives:
   ```yaml
   services:
      publisher:
         environment:
            NER_ENDPOINT: http://ner:8001
            CONTEXT_PATH: "" # TEI Publisher will be mapped to the root of the website
         build:
            context: https://github.com/my-github-user/my-edition.git#master
   ```
2. modify `conf/default.conf` and replace the two lines referring to `/apps/tei-publisher`:
   ```
   proxy_pass http://docker-publisher/exist/apps/my-edition$request_uri;
   proxy_redirect http://$host/exist/apps/my-edition/ /;
   ```
   For deployment on a public server you would need to apply the same to your copy of `conf/example.com.tmpl` (see above about how to copy/modify this file).

## 3. Start your compose config on a server

Rent a cloud server which has docker enabled. There are various offers on the market. A good specification would include 4 gb of RAM and 2 vCPU, which you can get for less than 10 Euro per month.

Once you have root access to your server, ssh into it and clone your customized docker compose configuration repository. Build and start the services as [described](#running). Depending on the configuration of your server, you may or may not have docker compose installed already. If it is not available (but docker itself is), follow the instructions in the [docker documentation](https://docs.docker.com/compose/install/).

Finally, acquire a SSL certificate following the steps [documented above](#deployment-on-a-public-server).