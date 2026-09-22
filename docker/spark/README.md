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

The Spark 4 images target `linux/amd64` and use Spark 4.2.0, Scala 2.13,
Hadoop 3.5.0, Temurin 25.0.4+7, and Python 3.11:

- `spark-app:4.2.0-scala2.13-java25-ubuntu-1` uses the digest-pinned Spark
  base configured by `SPARK4_BASE_IMAGE` in the Makefile.
- `spark-app-cuda:4.2.0-scala2.13-java25-ubuntu-1` copies Spark and Java from
  that base into the digest-pinned `SPARK4_CUDA_BASE_IMAGE` (CUDA 12.8.1,
  cuDNN 9.8.0.87-1).

These images do not include S3A, ABFS, GCS, or BigQuery connectors. They
therefore cannot access SeaweedFS through S3A. Connector support requires a
new immutable image revision, such as
`4.2.0-scala2.13-java25-ubuntu-2`.

MLRun selects the CPU repository and tag through `MLRUN_SPARK_APP_IMAGE` and
`MLRUN_SPARK_APP_IMAGE_TAG`, and derives the CUDA repository by appending
`-cuda`. mlefi recognizes that resulting `spark-app` / `spark-app-cuda` pair
using `^(.+?)-scala.*$`. These recipes do not change the configured defaults.

### Build

```bash
make build-spark4
make build-spark4-cuda
make build-spark4-all
```

`REGISTRY` defaults to `gcr.io/iguazio`. The Makefile is the source of truth
for both pinned base-image digests.

### Validate

```bash
make validate-spark4-all
```

The validators check Spark, Scala, Java, Hadoop, Python, image metadata, and
CUDA metadata. The aggregate target also checks CPU/CUDA environment parity
and compares every `$SPARK_HOME/jars` entry by SHA-256.

### Publish (manual, JFrog)

Publication is manual. First check that neither
`spark-app:4.2.0-scala2.13-java25-ubuntu-1` nor
`spark-app-cuda:4.2.0-scala2.13-java25-ubuntu-1` already exists in
`mckinsey-ig4-next-gen-docker-local.jfrog.io`. Never overwrite an existing
immutable tag.

```bash
make REGISTRY=mckinsey-ig4-next-gen-docker-local.jfrog.io build-spark4-all
make REGISTRY=mckinsey-ig4-next-gen-docker-local.jfrog.io validate-spark4-all

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
