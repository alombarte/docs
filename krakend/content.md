%%LOGO%%

# What is KrakenD?

[KrakenD](https://www.krakend.io/) is a stateless, high-performance, enterprise-ready, open source API gateway written in Go. Its engine (formerly known as *KrakenD Framework*) is now a **Linux Foundation Project** codenamed [Lura Project](https://luraproject.org/). Lura is the only enterprise-grade API Gateway hosted in a neutral, open forum.

KrakenD is very lightweight and simple to use as it only requires to write the configuration file, no Go knowledge is required. It offers  connectivity to internal and external services, transformation and filtering of data, aggregation of multiple data sources (APIs, gRPC, queues and pub/sub, lambda, etc) simultaneously or in cascade, protects the access to your API, throughputs its usage, and integrates with a lot of third-parties.

All features are designed to offer awesome performance and infinite scalability.

## How to use this image
KrakenD only needs a single configuration file to create an API Gateway, although you can have a complex setup reflecting your organization structure. The configuration file(s) can live anywhere in the container but the default location is `/etc/krakend`.

To use the image, `COPY` your `krakend.json` file inside the container, or mount it using a volume. The configuration is checked only once during the startup and never used again. Don't have a config file yet? Generate it with the [KrakenD Designer UI](https://designer.krakend.io).

⚠️ **NOTICE**: KrakenD does not use live reload when your configuration changes. Restart the container.

### Quick start
You can start an empty gateway with a health check with the following commands:

```console
$ echo '{ "version": 2 }' > krakend.json
$ docker run -d -p 8080:8080 -v "$PWD:/etc/krakend/" %%IMAGE%%
...
$ curl http://localhost:8080/__health
{"status":"ok"}
```



### More Examples
The following are several examples of running KrakenD. By default the command `run` is executed, but you can pass
other commands and flags at the end of the run command.

The configuration files are taken from current directory (`$PWD`). All examples expect to find at least the file `krakend.json`.

####  Run with the debug enabled (flag `-d`):
This flag is **SAFE to use in production**. It's meant to enable KrakenD as a fake backend itself by enabling a [`/__debug` endpoint](https://www.krakend.io/docs/endpoints/debug-endpoint/)

```console
    docker run -p 8080:8080 -v "${PWD}:/etc/krakend/" %%IMAGE%% run -d -c /etc/krakend/krakend.json
```

#### Checking the syntax of your configuration file
See the [check command](https://www.krakend.io/docs/commands/check/)

```console
    docker run -it -p 8080:8080 -v $PWD:/etc/krakend/ %%IMAGE%% check --config krakend.json
```
#### Show the help:

    docker run -it -p 8080:8080 -v $PWD:/etc/krakend/ %%IMAGE%% --help


### Building your custom KrakenD image

Most production deployments will not want to rely on mounting a volume for the container, but to use their own image based on `%%IMAGE%%`:

Your `Dockerfile` could look like this:

```Dockerfile
FROM %%IMAGE%%:<version>
# NOTE: Avoid using :latest image on production. Stick to a major version instead.

COPY krakend.json /etc/krakend/krakend.json

# Check and test that the file is valid
RUN krakend check -d -t -c /etc/krakend/krakend.json
```

If you want to manage your KrakenD configuration using multiple files and folders, reusing templates and distributing the configuration amongst your teams, you can use the [flexible configuration (FC)](https://www.krakend.io/docs/configuration/flexible-config/). The following `Dockerfile` combines FC, the `krakend check` command, and a 2-step build.

```Dockerfile
FROM %%IMAGE%%:<version> as builder
ARG ENV=prod

COPY krakend.tmpl .
COPY config .

# Save temporary output file to /tmp to avoid permission errors
RUN FC_ENABLE=1 \
    FC_OUT=/tmp/krakend.json \
    FC_PARTIALS="/etc/krakend/partials" \
    FC_SETTINGS="/etc/krakend/settings/$ENV" \
    FC_TEMPLATES="/etc/krakend/templates" \
    krakend check -d -t -c krakend.tmpl

# Copy the output file only and discard any other files
FROM %%IMAGE%%:<version>
COPY --from=builder --chown=krakend /tmp/krakend.json .
```
Then build with `docker build --build-arg ENV=prod -t my_krakend .`

The configuration above assumes you have a folder structure like the following:
```
.
├── config
│   ├── partials
│   ├── settings
│   │   ├── prod
│   │   │   └── env.json
│   │   └── test
│   │       └── env.json
│   └── templates
│       └── some.tmpl
├── Dockerfile
└── krakend.tmpl
```

### Docker Compose example
Finally, a simple `docker-compose` file to start KrakenD with your API would be:

```yaml
version: "3"
services:
  krakend:
    image: %%IMAGE%%:<version>
    ports:
      - "8080:8080"   
    volumes:
      - ./:/etc/krakend
```

And another one that uses flexible configuration and a custom template filename (`my_krakend.tmpl`) on each start:

```yaml
version: "3"
services:
  krakend:
    image: %%IMAGE%%:<version>
    ports:
      - "8080:8080"   
    volumes:
      - ./:/etc/krakend
    environment:
      - FC_ENABLE=1
      - FC_OUT=/tmp/krakend.json
      - FC_PARTIALS="/etc/krakend/config/partials"
      - FC_SETTINGS="/etc/krakend/config/settings/prod"
      - FC_TEMPLATES="/etc/krakend/config/templates"
    command:
      command: ["run", "-c", "krakend.tmpl", "-d"]
```
