# Flink Sample Application

> **DISCLAIMER:** This sample application is provided as-is, without warranty. It is intended to illustrate techniques for implementing various scenarios using Fast Data Platform, but it has not gone through a robust validation process, nor does it use all the techniques commonly employed for highly-resilient, production applications. Please use it with appropriate caution.

The sample application is adapted from the publicly-available [Flink training](http://dataartisans.github.io/flink-training/) from [dataArtisans](http://data-artisans.com/). It uses a public dataset of taxi rides in New York City. The details of the dataset can be found [here](http://dataartisans.github.io/flink-training/exercises/taxiData.html). This application will use [this](http://training.data-artisans.com/trainingData/nycTaxiRides.gz) data file. In summary, the application does the following:

1. Load the dataset using an application
2. Read the dataset from a Kafka topic (`taxiin`)
3. Analyze and transform the dataset
4. Train a regression classifier for travel time prediction
5. Predict travel time based on model in Step 3
6. Write the prediction to another Kafka topic (`taxiout`)

The main components of running the Flink sample application are:

1. Deploy the data ingestion module as a separate app, which will pull data from an S3 bucket and load it into a Kafka topic (`taxiin`)
2. Deploy travel time prediction application as a separate app, which will predict the travel time after training from the classifier and write the output into a Kafka topic (`taxiout`)

## Running the Applications Locally

All the applications can be run locally, or on OpenShift or Kubernetes.

`sbt` will be used to run applications on your local machine. The following examples demonstrate how to run the individual components from the `sbt` console.

### Running the Data Ingestion application

```
$ sbt
> projects
[info] In file:/Users/bucktrends/lightbend/fdp-sample-apps/flink/source/core/
[info] 	   fdp-flink-ingestion
[info] 	   fdp-flink-taxiride
[info] 	   ingestRun
[info] 	 * root
> project ingestRun
> ingest
```

This will run the data ingestion and transformation application on the local machine. Before running the application, please ensure the configuration files are set up appropriately for the local environment. Here's the default setup of `application.conf` within the `ingestion` folder of the project:

```
dcos {

  kafka {
    brokers = "localhost:9092,localhost:9093,localhost:9094"
    brokers = ${?KAFKA_BROKERS}

    group = "group"
    group = ${?KAFKA_GROUP}

    intopic = "taxiin"
    intopic = ${?KAFKA_IN_TOPIC}

    outtopic = "taxiout"
    outtopic = ${?KAFKA_OUT_TOPIC}

    zookeeper = "localhost:2181"
    zookeeper = ${?ZOOKEEPER_URL}

    ## settings for data ingestion
    loader {
      sourcetopic = ${dcos.kafka.intopic}
      sourcetopic = ${?KAFKA_IN_TOPIC}

      directorytowatch = "/Users/bucktrends/data"
      directorytowatch = ${?DIRECTORY_TO_WATCH}

      pollinterval = 1 second
    }
  }
}
```

All values can be set through environment variables as well. This is done when we deploy to a cluster. For local running just change the settings to the values of your local environment.

### Running the Taxi Ride Application

```
$ sbt
> projects
[info] In file:/Users/bucktrends/lightbend/fdp-sample-apps/flink/source/core/
[info] 	   fdp-flink-ingestion
[info] 	   fdp-flink-taxiride
[info] 	   ingestRun
[info] 	 * root
> project fdp-flink-taxiride
> run --broker-list localhost:9092 --inTopic taxiin --outTopic taxiout
```

## Deploying and Running on a Cluster

The first step in deploying the applications on a cluster is to prepare Docker images of all the applications. This can be done from within `sbt`.

### Prepare Docker Images

> **Note:** Prebuilt docker images are already available in Lightbend's Docker Hub account.

Building the app can be done using the convenient `build.sh` or `sbt`.

For `build.sh`, use one of the following commands:

```bash
build.sh
build.sh --push-docker-images
```

Both effectively run `sbt clean compile docker`, while the second variant also pushes the images to your Docker Hub account. _Only use this option_ if you first change `organization in ThisBuild := CommonSettings.organization` to `organization in ThisBuild := "myorg"` in `source/core/build.sbt`!

To use SBT, switch to the `flink/source/core/` directory and run the following commands:

```bash
$ sbt
> projects
[info] 	   fdp-flink-ingestion
[info] 	   fdp-flink-taxiride
[info] 	   ingestRun
[info] 	 * root
> docker
```

This will create Docker images named `lightbend/fdp-flink-ingestion:X.Y.Z` and `lightbend/fdp-flink-taxiride:X.Y.Z`, for the current version `X.Y.Z`.

The name of the docker user comes from the `organization` field in `build.sbt` and can be changed there for alternatives. You can use the `sbt` target `dockerPush` to push the images to Docker Hub, but only after changing the `organization` as just described. You can publish to your local (machine) repo with the `docker:publishLocal` target.

For IDE users, just import a project and use IDE commands.

If you want to build the two Docker images separately, use the `project` command in `sbt` to select which project, then run `docker`, e.g.,:

```bash
$ sbt
> project fdp-flink-ingestion
> docker
```

## Deploying and Running on OpenShift or Kubernetes

Use the Helm Chart in the `helm` directory to deploy the applications in a pod:

```bash
$ pwd
.../flink
$ helm install --name taxiride ./helm
...
$ kubectl logs <pod name where the application runs>
```

### Output of the Running Applications

The computation results for travel time prediction appears in the Kafka topic `taxiout`. You can run a consumer and check the predicted times as they flow across during processing of the application.

