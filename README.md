# apache-airflow-providers-pysparkonk8s
This is a Python package for the `pysparkonk8s` [Apache Airflow](https://airflow.apache.org) provider. All classes for this provider are in the 
`airflow.providers.pysparkonk8s` python package.

## Description

This provider package extends the default Airflow functionalities by introducing the `PySparkOnK8s` operator and 
`@task.pyspark_on_k8s` decorator that can both be used to run 
[PySpark](https://spark.apache.org/docs/latest/api/python/index.html) code as Airflow tasks on a Kubernetes-hosted 
Apache Airflow cluster.

The provider will automatically initialize an [Apache Spark](https://spark.apache.org/) cluster in either [`Client` or 
`Local` modes](https://spark.apache.org/docs/latest/submitting-applications.html) in an easily-configurable and 
transparent way to the user. [Spark Connect](https://spark.apache.org/docs/latest/spark-connect-overview.html) is also
supported.

### Running Spark in Client mode (default)
When running Spark in `Client` mode, the provider will provision a Spark cluster configuring Kubernetes as master, and 
will instantiate one Spark Driver pod and one or more Spark Executor pods depending on the configuration. 

The Spark Driver will coincide with the Airflow Worker pod running the task. This has several benefits, including a 
seamless integration with Airflow's [TaskFlow API](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html), 
and access to Airflow [Variables](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/variables.html), 
[Connections](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/connections.html) and 
[XComs](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/xcoms.html#xcoms) from within the PySpark 
code.

The Worker pod's requests and limits are dynamically mutated by the provider to match the Spark Driver's configuration. 
The Spark Driver will then provision the Executor pods by communicating directly with the Kubernetes API. Once the 
Airflow task is over, the Spark pods are destroyed and the relative provisioned resources are automatically released.

#### Spark Pod placement
To minimize networking bottlenecks and costs, Spark Executor Pods should be placed *close* to each other and possibly
*close* to the Spark Driver. Unless otherwise specified, the `pysparkonk8s` provider will generate pod affinity rules
to try to ensure the following pod placement:
1. try to spawn executors on the same node as the driver pod;
2. otherwise, try to spawn executors on the same node as the other executor pods;
3. otherwise, try to spawn executors in the same availability zone as the driver pod;
4. finally, try to spawn executors in the same availability zone as the other executor pods.

These affinity rules are generated dynamically on a per-task basis, which means that concurrent Airflow PySpark Tasks 
with their own Spark Drivers and Executors will not affect each-others pod placement.

### Running Spark in Local mode
In `Local` mode, Spark runs both Driver and Executor components on a single JVM. It is the simplest mode of deployment and 
is mostly used for testing and debugging. 

Similarly to the `Client` mode, the Airflow Worker pod's requests and limits are dynamically mutated by the provider to
match the provided Spark configuration. No Executor pods will be instantiated in this case.

### Running Spark in Connect mode
The provider also supports running PySpark code on an existing Spark Connect cluster. Please be aware that some Spark
functionalities [might not be supported](https://spark.apache.org/docs/latest/spark-connect-overview.html#what-is-supported-in-spark-34) 
when using Spark Connect.

## Usage
Several usage examples are available in the `examples/dags/` folder. Documentation on how to setup a local testing 
environment with minikube are available [here](examples/README.md).

## Requirements
The provider requires Apache Airflow v2.6.0 or later.

The provider requires the Airflow deployment to be configured with a Kubernetes-based 
[executor](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/executor/index.html), i.e. one of the 
following:
* [Kubernetes Executor](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/executor/kubernetes.html)
* [CeleryKubernetes Executor](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/executor/celery_kubernetes.html)
* [LocalKubernetes Executor](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/executor/local_kubernetes.html)

To allow the creation of Executor pods in Kubernetes clusters with RBAC enabled, it is essential to configure the 
the Spark Driver/Airflow Worker pod's service account with an appropriate role. 
By default, the Spark Driver/Airflow Worker pod is assigned a Kubernetes service account named `airflow-worker`. This 
service account is used to interact with the Kubernetes API server for the creation and monitoring of executor pods. 
The required role and role binding can be set up using the `pysparkonk8s-addon` Helm Chart available in this repository.

For an exhaustive list of dependencies please refer to the `pyproject.toml` file.

**Note:** This project was tested with Python version 3.10 and Apache Airflow version 2.8.1. 

## Installation
We set the following environment variables that will be used throughout the environment setup.

```shell
export PYTHON_VERSION="3.10"
export AIRFLOW_VERSION="2.8.1"
```

The provided package can be installed in a reproducible way with the following command:
```shell
pip install apache-airflow==${AIRFLOW_VERSION} \
  git+https://github.com/sebastiandaberdaku/apache-airflow-providers-pysparkonk8s.git@main \
  --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"
```
or (if git is missing):
```shell
pip install apache-airflow==${AIRFLOW_VERSION} \
  https://github.com/sebastiandaberdaku/apache-airflow-providers-pysparkonk8s/archive/refs/heads/main.tar.gz \
  --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"
```

Usually, after completing the reproducible installation, you can incorporate additional dependencies and providers by 
executing separate commands. This approach allows you the flexibility to upgrade or downgrade dependencies as needed, 
without being constrained by specific limitations. A recommended practice is to extend the pip install command with the 
pinned version of apache-airflow that you have already installed. This precaution ensures that apache-airflow is not 
inadvertently upgraded or downgraded by pip during the process.

### Building Apache Airflow Docker images with the `pysparkonk8s` provider
An example of how to build a Docker image with the `pysparkonk8s` provider is available in the `docker/` folder. 
See the related [documentation](docker/README.md) for more details.

### Installing `pysparkonk8s-addon`
The `chart/` folder contains the `pysparkonk8s-addon` Helm Chart that installs the required role and role-binding 
for the provider. 

The `docs/` folder contains the compiled Helm package and serves it as a self-hosted Helm repository via GitHub Pages.
See the related [documentation](docs/README.md) for more details.

To install the addon in the `airflow` namespace use the following command:
```shell
helm repo add pysparkonk8s https://sebastiandaberdaku.github.io/apache-airflow-providers-pysparkonk8s
helm upgrade --install pysparkonk8s pysparkonk8s/pysparkonk8s-addon --namespace airflow
```

## Testing the provider locally 
### Prerequisites
Ensure that your testing environment has:
* Airflow 2.6.0 or later. You can check your version by running `airflow version`.
* All provider packages that your DAG uses.
* An initialized Airflow metadata database, if your DAG uses elements of the metadata database like XComs. The Airflow 
metadata database is created when Airflow is first run in an environment. You can check that it exists with `airflow db 
check` and initialize a new database with `airflow db migrate` (`airflow db init` in Airflow versions pre-2.7).

### Cloning the project from GitHub
From a terminal run the following command to clone the project and navigate into the newly created folder:
```shell
git clone https://github.com/sebastiandaberdaku/apache-airflow-providers-pysparkonk8s.git
cd apache-airflow-providers-pysparkonk8s
```

### Installing required testing dependencies
The project dependencies for testing are provided in the `requirements-dev.txt` file. We recommend creating a dedicated 
[Conda environment](https://conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html) for managing 
the dependencies.

Setup and activate Conda environment with the following command:
```shell
conda create --name pysparkonk8s python=${PYTHON_VERSION} -y
conda activate pysparkonk8s
```

Install the current package and its dependencies with the following command launched from the root of the project:
```shell
pip install apache-airflow==${AIRFLOW_VERSION} \
  . \
  --requirement requirements-dev.txt \
  --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"
```

The provider can also be installed in editable mode by providing the `--editable` flag:
```shell
pip install apache-airflow==${AIRFLOW_VERSION} \
  --editable . \
  --requirement requirements-dev.txt \
  --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"
```

### Set up a Database Backend
Airflow requires a Database Backend to manage its metadata. In production environments, you should consider setting up a 
database backend to PostgreSQL, MySQL, or MSSQL. By default, Airflow uses SQLite, which is intended for development 
purposes only.

For testing purposes SQLite can be used. To initialize the SQLite database please execute the following commands:
```shell
export AIRFLOW__CORE__LOAD_EXAMPLES=False
export AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=sqlite:////tmp/airflow.db
airflow db migrate
```
This will create the `airflow.db` file in your current folder. This file is already included in `.gitignore`, however 
please make sure you are not accidentally adding it to git if you change the default file path.

If you have any issues in setting up the SQLite database please refer to the 
[official Airflow documentation](https://airflow.apache.org/docs/apache-airflow/stable/howto/set-up-database.html#setting-up-a-sqlite-database).

### Running the tests
The tests (available in the `tests/` folder) can be run with the following commands.

```shell
export AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=sqlite:////tmp/airflow.db
export AIRFLOW_HOME=airflow_home
pytest tests/
```