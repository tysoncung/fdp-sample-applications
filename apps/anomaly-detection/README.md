# Anomaly Detection using Deep Learning

> **DISCLAIMER:** This sample application is provided as-is, without warranty. It is intended to illustrate techniques for implementing various scenarios using Fast Data Platform, but it has not gone through a robust validation process, nor does it use all the techniques commonly employed for highly-resilient, production applications. Please use it with appropriate caution.

This application demonstrates a full lifecycle of anomaly detection using supervised learning developed using deep learning and leveraging [Intel BigDL](https://software.intel.com/en-us/articles/bigdl-distributed-deep-learning-on-apache-spark) as the implementation framework.

The application consists of the following modules:

* **Data publisher:** Generates simulated CPU signal data based on a probability distribution and publishes to Kafka
* **Data collector:** Collects data from Kafka and stores them in InfluxDB for use by machine learning and visualization
* **Data ingester:** Ingests data from InfluxDB and prepares them in CSV format for model training
* **Model training:** Trains a deep learning model
* **Model publisher:** Publishes the trained model and some useful statictics to Kafka to be used by the model server
* **Model serving:** Serving models in real time and updating them as new models will become available. It's based on the
[minibook](https://www.lightbend.com/blog/serving-machine-learning-models-free-oreilly-ebook-from-lightbend). This module is here for demonstration purposes only. It is obsolete and not in use any more.
* **Speculative model serving:** Serving models, leveraging speculative execution and picking results by _voting_. Models are updated as new models will become available. Based on this [blog post](https://developer.lightbend.com/blog/2018-05-24-speculative-model-serving/index.html)

# Installation

## Installing Data Publisher and Speculative model server

These two modules can be installed either on DC/OS or Kubernetes on DC/OS.

For installation on DC/OS use this [json configuration](/apps/anomaly-detection/source/core/adpublisher/src/main/resources/adpublisher.json)
for publisher and  [this one](/apps/anomaly-detection/source/core/adspeculativemodelserver/src/main/resources/adspeculativemodelserver.json) for speculative model server.

On OpenShift and Kubernetes, these two modules are installed as 2 pods using the enclosed [Helm Chart](/apps/anomaly-detection/helm).

Make sure you update the [values.yaml](/apps/anomaly-detection/helm/values.yaml) to match your environment. The definitions for the values can be found [here](/apps/anomaly-detection/helm/values-metadata.yaml).

## Installing All Training Modules

The current version of the training modules is supported only on DC/OS.

The three training modules that need to be installed are:

* Data ingester (pre-training)
* Model training
* Model publisher (post-training)

All of the above modules are installed on a DC/OS pod as 3 images. The JSON that has to be used for deployment is located in `.../fdp-sample-applications/apps/anomaly-detection/bin`. It can be deployed using the command `dcos marathon pod add training-pod.json`.

> This installation procedure assumes that the previous 2 installation steps have been completed and all prerequisites (Kafka and InfluxDB) running in the cluster.

### Preparing All the Docker Images at Once

Building all the app Docker images can be done using the convenient `build.sh` or `sbt`.

For `build.sh`, use one of the following commands:

```bash
build.sh
build.sh --push-docker-images
```

Both effectively run `sbt clean compile docker` for each nested project, while the second variant also pushes the images to your Docker Hub account. _Only use this option_ if you first change `organization in ThisBuild := CommonSettings.organization` to `organization in ThisBuild := "myorg"` in `source/core/build.sbt`!

For `sbt`, the steps are as follows:

```bash
$ cd source/core
$ pwd
.../fdp-sample-applications/apps/anomaly-detection/source/core
$ sbt
sbt:AnomalyDetection> projects
[info] In file: .../fdp-sample-applications/apps/anomaly-detection/
[info]      fdp-ad-model-server
[info]      fdp-ad-data-publisher
[info]      fdp-ad-speculative-model-server
[info]    * anomalyDetection
[info]      configuration
[info]      influxSupport
[info]      kafkaSupport
[info]      model
[info]      protobufs
[info]      fdp-ad-training-data-ingestion
[info]      fdp-ad-trainingmodel-publish
sbt:AnomalyDetection> docker
...
```

You can use the `sbt` target `dockerPush` to push the images to Docker Hub, but only after changing the `organization` as just described. You can publish to your local (machine) repo with the `docker:publishLocal` target.

For IDE users, just import a project and use IDE commands, but it is necessary to run `sbt clean compile` at least once to compile the `protobufs` subproject correctly.

### Preparing the Docker Image for the Data Ingester

If you want to work with the nested projects individually, use `sbt`:

```bash
$ cd source/core
$ pwd
.../fdp-sample-applications/apps/anomaly-detection/source/core
$ sbt
sbt:AnomalyDetection> projects
[info] In file: .../fdp-sample-applications/apps/anomaly-detection/
[info] 	   fdp-ad-model-server
[info] 	   fdp-ad-data-publisher
[info] 	   fdp-ad-speculative-model-server
[info] 	 * anomalyDetection
[info] 	   configuration
[info] 	   influxSupport
[info] 	   kafkaSupport
[info] 	   model
[info] 	   protobufs
[info] 	   fdp-ad-training-data-ingestion
[info] 	   fdp-ad-trainingmodel-publish
sbt:AnomalyDetection> project fdp-ad-training-data-ingestion
[info] Set current project to fdp-ad-training-data-ingestion (in build file: ..)
sbt:AnomalyDetection> docker
...
```

You can use the `sbt` target `dockerPush` to push the images to Docker Hub, as described in the previous section.

### Preparing the Docker Image for Model Training

The Docker image for model training uses Intel's Analytics Zoo and the image is also derived from the base image created by Intel. The detailed steps of how to build the Docker image are specified in the `README` file under `.../fdp-sample-applications/apps/anomaly-detection/analytics-zoo/apps/python/lightbend`.

### Preparing the Docker Image for the Model Publisher

If you want to work with the nested projects individually, use `sbt`:

```bash
$ pwd
.../fdp-sample-applications/apps/anomaly-detection
$ sbt
sbt:AnomalyDetection> projects
[info] In file: .../fdp-sample-applications/apps/anomaly-detection/source/core
[info] 	   fdp-ad-model-server
[info] 	   fdp-ad-data-publisher
[info] 	   fdp-ad-speculative-model-server
[info] 	 * anomalyDetection
[info] 	   configuration
[info] 	   influxSupport
[info] 	   kafkaSupport
[info] 	   model
[info] 	   protobufs
[info] 	   fdp-ad-training-data-ingestion
[info] 	   fdp-ad-trainingmodel-publish
sbt:AnomalyDetection> project fdp-ad-training-model-publish
[info] Set current project to fdp-ad-training-model-publish (in build file: ..)
sbt:AnomalyDetection> docker
...
```

You can use the `sbt` target `dockerPush` to push the images to Docker Hub, as described in a previous section.
