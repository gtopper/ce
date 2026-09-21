# CE Spark images

This directory builds:

- `gcr.io/iguazio/spark-app:3.5.6-scala2.12-java17-ubuntu-1` (`Dockerfile`) --
  the regular CE Spark image.
- `gcr.io/iguazio/spark-app-cuda:3.5.6-scala2.12-java17-ubuntu-1` (`Dockerfile.cuda`) --
  the same Spark distribution and CE customization on CUDA 12.8.1/cuDNN 9.8.

MLRun selects the CUDA image by appending `-cuda` to the configured repository.
Both images share `scripts/ce-customize.sh`.

## Build

```bash
make build
make build-cuda
make build-all
```

Override `MLRUN_CE_SPARK_IMAGE_TAG`, `MLRUN_CE_SPARK_CUDA_IMAGE_TAG`,
`MLRUN_CE_IMAGE_PLATFORM`, or `CUDA_VERSION` as needed.

## Validate

These checks verify image contents and metadata without requiring a GPU.
GPU visibility and Spark GPU scheduling require a GPU environment.

```bash
make validate
make validate-cuda
make validate-all
```

## Publish

Build and validate the CUDA image, then push it manually:

```bash
make build-cuda
make validate-cuda

gcloud auth configure-docker gcr.io
docker push gcr.io/iguazio/spark-app-cuda:3.5.6-scala2.12-java17-ubuntu-1

docker inspect --format '{{index .RepoDigests 0}}' \
  gcr.io/iguazio/spark-app-cuda:3.5.6-scala2.12-java17-ubuntu-1
```

Do not republish the existing regular image. Record the CUDA image digest and
the CE source commit.

## Spark 4

The Spark 4 images use Spark 4.2.0, Scala 2.13, Hadoop 3.5.0, Temurin
25.0.4+7, and Python 3.11:

- `spark-app:4.2.0-scala2.13-java25-ubuntu-1` uses
  `spark@sha256:66e39dccde81909c23e5c56f4b465db6de2bdc38cf569aacc9e2380ee5005440`.
- `spark-app-cuda:4.2.0-scala2.13-java25-ubuntu-1` copies Spark and Java from
  that image into
  `nvidia/cuda@sha256:61f6c08f2b59036cb935e56d1e31a6b64e3ae2c7ddb86d33fa0b044c7917b719`
  (CUDA 12.8.1, cuDNN 9.8.0.87-1).

These images do not include S3A, ABFS, GCS, or BigQuery connectors. They
therefore cannot access SeaweedFS through S3A. Connector support requires a
new immutable image revision, such as
`4.2.0-scala2.13-java25-ubuntu-2`.

MLRun selects the repository and tag through `MLRUN_SPARK_APP_IMAGE` and
`MLRUN_SPARK_APP_IMAGE_TAG`. mlefi resolves the `spark-app` /
`spark-app-cuda` pair using `^(.+?)-scala.*$`. These recipes do not change
the configured defaults.

### Build

```bash
make build-spark4
make build-spark4-cuda
make build-spark4-all
```

`REGISTRY` defaults to `gcr.io/iguazio`. Override it or either image-tag
variable as needed.

### Validate

```bash
make validate-spark4-all
make jar-parity-spark4
```

The validators check Spark, Scala, Java, Hadoop, Python, image metadata, and
CUDA metadata. The parity check compares every `$SPARK_HOME/jars` entry by
SHA-256.

### Publish (manual, JFrog)

Publication is manual. First check that neither
`spark-app:4.2.0-scala2.13-java25-ubuntu-1` nor
`spark-app-cuda:4.2.0-scala2.13-java25-ubuntu-1` already exists in
`mckinsey-ig4-next-gen-docker-local.jfrog.io`. Never overwrite an existing
immutable tag.

```bash
make REGISTRY=mckinsey-ig4-next-gen-docker-local.jfrog.io build-spark4-all
make REGISTRY=mckinsey-ig4-next-gen-docker-local.jfrog.io validate-spark4-all
make REGISTRY=mckinsey-ig4-next-gen-docker-local.jfrog.io jar-parity-spark4

docker login mckinsey-ig4-next-gen-docker-local.jfrog.io
docker push mckinsey-ig4-next-gen-docker-local.jfrog.io/spark-app:4.2.0-scala2.13-java25-ubuntu-1
docker push mckinsey-ig4-next-gen-docker-local.jfrog.io/spark-app-cuda:4.2.0-scala2.13-java25-ubuntu-1

docker inspect --format '{{index .RepoDigests 0}}' \
  mckinsey-ig4-next-gen-docker-local.jfrog.io/spark-app:4.2.0-scala2.13-java25-ubuntu-1
docker inspect --format '{{index .RepoDigests 0}}' \
  mckinsey-ig4-next-gen-docker-local.jfrog.io/spark-app-cuda:4.2.0-scala2.13-java25-ubuntu-1
```

Record both repository digests, then pull each image back by digest and
rerun validation against the digest references. A local image ID is not a
published digest.

### Jira evidence (ML-13080)

Attach to ML-13080:

- immutable tags and repository digests;
- pull-by-digest and image-inspection output;
- `spark-submit --version` and `java -version` output;
- CPU and CUDA JAR SHA-256 inventories and the parity result;
- the source commit;
- the connector limitations documented above.
