# Telemetry-Airflow

[![CircleCi](https://circleci.com/gh/mozilla/telemetry-airflow.svg?style=shield&circle-token=62f4c1be98e5c9f36bd667edb7545fa736eed3ae)](https://circleci.com/gh/mozilla/telemetry-airflow)

[Apache Airflow](https://airflow.apache.org/) is a platform to programmatically
author, schedule and monitor workflows.

This repository codifies the Airflow cluster that is deployed at
[workflow.telemetry.mozilla.org](https://workflow.telemetry.mozilla.org)
(behind SSO) and commonly referred to as "WTMO" or simply "Airflow".

Some links relevant to users and developers of WTMO:

- The `dags` directory in this repository contains some custom DAG definitions
- Many of the DAGs registered with WTMO don't live in this repository, but are
  instead generated from ETL task definitions in
  [bigquery-etl](https://github.com/mozilla/bigquery-etl)
- The Data SRE team maintains a
  [WTMO Developer Guide](https://mana.mozilla.org/wiki/display/DOPS/WTMO+Developer+Guide)
  (behind SSO)

## Writing DAGs
See the Airflow's [Best Practices guide](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html#best-practices) to help you write DAGs.

**⚠ Note: How to import DAGs and modules ⚠** 

**Modules should be imported from the project directory, such as `from dags.my_dag import load_data` 
rather than `from my_dag import load_data`.**

In Airflow, the `dags`, `config`, and `plugins` folders are [automatically added](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/modules_management.html#built-in-pythonpath-entries-in-airflow) to the 
`PYTHONPATH` to ensure they can be imported and accessed by Airflow's execution environment. 

However, this default configuration can cause problems when running unit tests located 
in the `tests` directory. Since the `PYTHONPATH` includes the `dags` directory, 
but not the project directory itself, the unit tests will not be able to import code from 
the `dags` directory. This limitation restricts the ability to test the DAGs effectively 
within the project structure. It is also generally expected that imports should work from the 
project directory rather than from any of its subdirectories. For this reason, `telemetry-airflow`'s 
Dockerfile adds the project directory to `PYTHONPATH`.

## Prerequisites

This app is built and deployed with
[docker](https://docs.docker.com/) and
[docker-compose](https://docs.docker.com/compose/).
Dependencies are managed with
[pip-tools](https://pypi.org/project/pip-tools/) `pip-compile`.

You'll also need to install PostgreSQL to build the database container.

### Installing dependencies locally
**_⚠ Make sure you use the right Python version. Refer to Dockerfile for current supported Python Version ⚠_**

You can install the project dependencies locally to run tests with Pytest. We use the 
[official Airflow constraints](https://airflow.apache.org/docs/apache-airflow/stable/installation/installing-from-pypi.html#constraints-files) file to simplify 
Airflow dependency management. Install dependencies locally using the following command:

```bash
make pip-install-local
```

### Updating Python dependencies

Add new Python dependencies into `requirements.in` and execute `make pip-install-local`

### Build Container

Build Airflow image with

```bash
make build
```
### Local Deployment
To deploy the Airflow container on the docker engine, with its required dependencies, run:

```bash
make build
make up
```
#### macOS
Assuming you're using Docker for Docker Desktop for macOS, start the docker service,
click the docker icon in the menu bar, click on preferences and change the
available memory to 4GB.

## Testing

### Adding dummy credentials

Tasks often require credentials to access external credentials. For example, one may choose to store
API keys in an Airflow connection or variable. These variables are sure to exist in production but
are often not mirrored locally for logistical reasons. Providing a dummy variable is the preferred
way to keep the local development environment up to date.

Update the `resources/dev_variables.env` and `resources/dev_connections.env` with appropriate strings to
prevent broken workflows.


### Usage

You can now connect to your local Airflow web console at
`http://localhost:8080/`.

All DAGs are paused by default for local instances and our staging instance of Airflow.
In order to submit a DAG via the UI, you'll need to toggle the DAG from "Off" to "On".
You'll likely want to toggle the DAG back to "Off" as soon as your desired task starts running.


### Testing GKE Jobs (including BigQuery-etl changes)

See https://go.corp.mozilla.com/wtmodev for more details.

```
make build && make up
make gke

When done:
make clean-gke
```

From there, [connect to Airflow](localhost:8080) and enable your job.

### Testing Dataproc Jobs

Dataproc jobs run on a self-contained Dataproc cluster, created by Airflow.

To test these, jobs, you'll need a sandbox account and corresponding service account.
For information on creating that, see "Testing GKE Jobs". Your service account
will need Dataproc and GCS permissions (and BigQuery, if you're connecting to it). _Note_: Dataproc requires "Dataproc/Dataproc Worker"
as well as Compute Admin permissions.
You'll need to ensure that the Dataproc API is [enabled in your sandbox project.](https://console.developers.google.com/apis/api/dataproc.googleapis.com)

Ensure that your dataproc job has a configurable project to write to.
Set the project in the DAG entry to be configured based on development environment;
see the `ltv.py` job for an example of that.

From there, run the following:

```bash
make build && make up
./bin/add_gcp_creds $GOOGLE_APPLICATION_CREDENTIALS google_cloud_airflow_dataproc
```

You can then connect to Airflow [locally](localhost:8080). Enable your DAG and see that it runs correctly.

### Production Setup
This repository was structured to be deployed using the [offical Airflow Helm Chart.](https://airflow.apache.org/docs/helm-chart/stable/index.html).
See the [Production Guide](https://airflow.apache.org/docs/helm-chart/stable/production-guide.html) for best practices.

### Debugging

Some useful docker tricks for development and debugging:

```bash
make clean

# Remove any leftover docker volumes:
docker volume rm $(docker volume ls -qf dangling=true)

# Purge docker volumes (helps with postgres container failing to start)
# Careful as this will purge all local volumes not used by at least one container.
docker volume prune
```
