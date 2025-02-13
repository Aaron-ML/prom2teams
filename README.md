![Logo](https://raw.githubusercontent.com/idealista/prom2teams/master/logo.gif)

[![Build Status](https://travis-ci.org/idealista/prom2teams.png)](https://travis-ci.org/idealista/prom2teams) 
[![Docker Build Status](https://img.shields.io/docker/build/idealista/prom2teams.svg)](https://hub.docker.com/r/idealista/prom2teams/) 
[![Docker Automated build](https://img.shields.io/docker/automated/idealista/prom2teams.svg)](https://hub.docker.com/r/idealista/prom2teams/)

# prom2teams

<img src="https://raw.githubusercontent.com/idealista/prom2teams/master/assets/example.png" alt="Alert example" style="width: 600px;"/>

**prom2teams** is a Web server built with Python that receives alert notifications from a previously configured [Prometheus Alertmanager](https://github.com/prometheus/alertmanager) instance and forwards it to [Microsoft Teams](https://teams.microsoft.com/) using defined connectors.

- [Getting Started](#getting-started)
	- [Prerequisities](#prerequisites)
	- [Installing](#installing)
- [Usage](#usage)
  - [Config file](#config-file)
	- [Configuring Prometheus](#configuring-prometheus)
	- [Templating](#templating)
- [Testing](#testing)
- [Built With](#built-with)
- [Versioning](#versioning)
- [Authors](#authors)
- [License](#license)
- [Contributing](#contributing)

## Getting Started

### Prerequisites

The application has been tested with _Prometheus 2.2.1_, _Python 3.5.0_ and _pip 9.0.1_.

Newer versions of _Prometheus/Python/pip_ should work but could also present issues.

### Installing

prom2teams is present on [PyPI](https://pypi.python.org/pypi/prom2teams), so could be installed using pip3:

```bash
$ pip3 install prom2teams
```

**Note:** Works since v1.1.1

## Usage

**Important:** Config path must be provided with at least one Microsoft Teams Connector. Check the options to know how you can supply it.

```bash
# To start the server (enable metrics, config file path , group alerts by, log file path, log level and Jinja2 template path are optional arguments):
$ prom2teams [--enablemetrics] [--configpath <config file path>] [--groupalertsby ("name"|"description"|"instance"|"severity"|"summary")] [--logfilepath <log file path>] [--loglevel (DEBUG|INFO|WARNING|ERROR|CRITICAL)] [--templatepath <Jinja2 template file path>]

# To show the help message:
$ prom2teams --help
```
Other options to start the service are:

```bash
export APP_CONFIG_FILE=<config file path>
$ prom2teams
```
**Note:** Grouping alerts works since v2.2.1

### Prom2teams Prometheus metrics

Prom2teams uses Flask and, to have the service monitored, we use @rycus66's [Prometheus Flask Exporter](https://github.com/rycus86/prometheus_flask_exporter). This will enable an endpoint in `/metrics` where you could find interesting metrics to monitor such as number of responses with a certain status. To enable this endpoint, just either:

- Use the `--enablemetrics` or `-m` flag when launching prom2teams.
- Set the environment variable `PROM2TEAMS_PROMETHEUS_METRICS=true`.

### Docker image

Every new Prom2teams release, a new Docker image is built in our [Dockerhub](https://hub.docker.com/r/idealista/prom2teams). We strongly recommend you to use the images with the version tag, though it will be possible to use them without it.

There are two things you need to bear in mind when creating a Prom2teams container:

- The connector URL must be passed as the environment variable `PROM2TEAMS_CONNECTOR`
- In case you want to group alerts, you need to pass the field as the environment variable `PROM2TEAMS_GROUP_ALERTS_BY`
- You need to map container's Prom2teams port to one on your host.

So a sample Docker run command would be:

```bash
$ docker run -it -d -e PROM2TEAMS_GROUP_ALERTS_BY=FIELD_YOU_WANT_TO_GROUP_BY -e PROM2TEAMS_CONNECTOR="CONNECTOR_URL" -p 8089:8089 idealista/prom2teams:VERSION
```

#### Provide custom config file

If you prefer to use your own config file, you just need to provide it as a Docker volume to the container and map it to `/opt/prom2teams/config.ini`. Sample:

```bash
$ docker run -it -d -v pathToTheLocalConfigFile:/opt/prom2teams/config.ini -p 8089:8089 idealista/prom2teams:VERSION
```

### Production

For production environments you should prefer using a WSGI server. [uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/)
dependency is installed for an easy usage. Some considerations must be taken to use it:

The binary `prom2teams_uwsgi` launches the app using the uwsgi server. Due to some incompatibilities with [wheel](https://github.com/pypa/wheel)
you must install `prom2teams` using `sudo pip install --no-binary :all: prom2teams` (https://github.com/pypa/wheel/issues/92)

```bash
$ prom2teams_uwsgi <path to uwsgi ini config>
```

And `uwsgi` would look like:

```
[uwsgi]
master = true
processes = 5
#socket = 0.0.0.0:8001
#protocol = http
socket = /tmp/prom2teams.sock
chmod-socket = 777
vacuum = true
env = APP_ENVIRONMENT=pro
env = APP_CONFIG_FILE=/etc/default/prom2teams.ini
```

Consider not provide `chdir` property neither `module` property.

Also you can set the `module` file, by doing a symbolic link: `sudo mkdir -p /usr/local/etc/prom2teams/ && sudo ln -sf /usr/local/lib/python3.5/dist-packages/usr/local/etc/prom2teams/wsgi.py /usr/local/etc/prom2teams/wsgi.py` (check your dist-packages folder)

Another approach is to provide yourself the `module` file [module example](bin/wsgi.py) and the `bin` uwsgi call [uwsgi example](bin/prom2teams_uwsgi)

**Note:** default log level is DEBUG. Messages are redirected to stdout. To enable file log, set the env APP_ENVIRONMENT=(pro|pre)


### Config file

The config file is an [INI file](https://docs.python.org/3/library/configparser.html#supported-ini-file-structure) and should have the structure described below:

```
[Microsoft Teams]
# At least one connector is required here
Connector: <webhook url>
AnotherConnector: <webhook url>   
...

[HTTP Server]
Host: <host ip> # default: localhost
Port: <host port> # default: 8089

[Log]
Level: <loglevel (DEBUG|INFO|WARNING|ERROR|CRITICAL)> # default: DEBUG
Path: <log file path>  # default: /var/log/prom2teams/prom2teams.log

[Template]
Path: <Jinja2 template path> # default: app resources template

[Group Alerts]
Field: <Field to group alerts by> # alerts won't be grouped by default

[Labels]
Excluded: <Coma separated list of labels to ignore>
```

**Note:** Grouping alerts works since v2.2.0

### Configuring Prometheus

The [webhook receiver](https://prometheus.io/docs/alerting/configuration/#<webhook_config>) in Prometheus allows configuring a prom2teams server.

The url is formed by the host and port defined in the previous step.

**Note:** In order to keep compatibility with previous versions, v2.0 keep attending the default connector ("Connector") in the endpoint 0.0.0.0:8089. This will be removed in future versions.   

```
// The prom2teams endpoint to send HTTP POST requests to.
url: 0.0.0.0:8089/v2/<Connector1>
```

### Templating

prom2teams provides a [default template](prom2teams/resources/templates/teams.j2) built with [Jinja2](http://jinja.pocoo.org/docs/2.10/) to render messages in Microsoft Teams. This template could be overrided using the 'templatepath' argument ('--templatepath <Jinja2 template file path>') during the application start.

Some fields are considered mandatory when received from Alert Manager.
If such a field is not included a default value of 'unknown' is assigned.

All non-mandatory fields and not in excluded list are injected in `extra_labels` key.

#### Swagger UI

Accessing to `<Host>:<Port>` (e.g. `localhost:8089`) in a web browser shows the API v1 documentation.

<img src="https://raw.githubusercontent.com/idealista/prom2teams/master/assets/swagger_v1.png" alt="Swagger UI" style="width: 600px;"/>

Accessing to `<Host>:<Port>/v2` (e.g. `localhost:8089/v2`) in a web browser shows the API v2 documentation.

<img src="https://raw.githubusercontent.com/idealista/prom2teams/master/assets/swagger_v2.png" alt="Swagger UI" style="width: 600px;"/>

## Testing

To run the test suite you should type the following:

```bash
// After cloning prom2teams :)
$ python3 -m unittest discover tests
```

## Built With
![Python 3.6.2](https://img.shields.io/badge/Python-3.6.2-green.svg)
![pip 9.0.1](https://img.shields.io/badge/pip-9.0.1-green.svg)

## Versioning

For the versions available, see the [tags on this repository](https://github.com/idealista/prom2teams/tags).

Additionaly you can see what change in each version in the [CHANGELOG.md](CHANGELOG.md) file.

## Authors

* **Idealista** - *Work with* - [idealista](https://github.com/idealista)

See also the list of [contributors](https://github.com/idealista/prom2teams/contributors) who participated in this project.

## License

![Apache 2.0 License](https://img.shields.io/hexpm/l/plug.svg)

This project is licensed under the [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) license - see the [LICENSE](LICENSE) file for details.

## Contributing

Please read [CONTRIBUTING.md](.github/CONTRIBUTING.md) for details on our code of conduct, and the process for submitting pull requests to us.
