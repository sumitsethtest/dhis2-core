
### General idea

The `dhis2-core/Dockerfile` handles building DHIS2 from source, and then
leaves the `dhis.war` available at `/dhis.war`, along with `CHECKSUM`
files..

This base image can be tagged as necessary and be reused across multiple
other assemblies at this point.

See [this
comment](https://github.com/dhis2/dhis2-core/pull/3894#issuecomment-539416233)
for container image build times.

### File structure

```
./dhis2-core
├ README.md
├ dhis-2
├ docker
│   ├ build-containers.sh
│   ├ extract-artifacts.sh
│   ├ shared
│   │   └ server.xml
│   ├ tomcat-alpine
│   │   ├ docker-entrypoint.sh
│   │   └ Dockerfile
│   └ tomcat-debian
│       ├ docker-entrypoint.sh
│       └ Dockerfile
├ Dockerfile
└ Jenkinsfile-publish-image
```

### Usage

#### Build the core image (a.k.a. the base image)

```sh
docker build -t <image> .
```

#### Extract artifacts from base image

The extract script accepts an image from which to extract known
artifacts from as first positional parameter.

```sh
./docker/extract-artifacts.sh <image>
```

The script dumps the artifacts in `./docker/artifacts`:

```
./dhis2-core/docker/artifacts
├ dhis.war
├ sha256sum.txt
└ md5sum.txt
```

The `extract-artifacts` _always_ purges the `artifacts` directory to
ensure consistency.
####

Customize which containers will be built and published using the
`shared/containers-list.sh` file:
https://github.com/dhis2/dhis2-core/pull/3894/files#diff-553ea93c842ece0c53a6331741a73e37

#### Build container images (Tomcat)

> :information_source: Uses `containers-list.sh` to get a list of containers to build

```sh
./docker/build-containers.sh <image>
```

Given that `image` is `core:local` the result is:

```
REPOSITORY          TAG                           IMAGE ID
CREATED             SIZE
core                local-8.5.34-jre8-alpine      cc5bcfc1964c        4
seconds ago       372MB
core                local-8.0-jre8-slim           d9d8ad7d7352        18
seconds ago      506MB
core                local-8.5-jdk8-openjdk-slim   77979af6aaab        41
seconds ago      587MB
core                local-9.0-jdk8-openjdk-slim   35f234ab3688        59
seconds ago      588MB
core                local                         db4a67b2ca73        55
minutes ago      270MB
```

#### Build custom WAR file into containers

The `build-containers` script will pick up and use any `dhis.war` from
the `dhis2-core/docker/artifacts` directory by default. This can be used
to create containers that contain a custom WAR-file.

> :warning: To use a custom `dhis.war` the checksum files must be removed (skips
> validation), or proper ones need to be generated.

E.g. how to build a custom `dhis.war` into the containers:

```sh
mv ~/my-custom-dhis.war ./docker/artifacts/dhis.war

./docker/build-containers.sh core:local-test
```

#### Publish container images (Tomcat)

> :information_source: Uses `containers-list.sh` to get a list of containers to publish

```sh
./docker/publish-containers.sh <image>
```

#### Cleanup images

> :information_source: Uses `containers-list.sh` to get a list of
> containers to clean up

```sh
./docker/cleanup-images.sh <image>
```

#### Checksums

Checksums are now generated for the build artifact and stored alongside
the WAR-file in sha256 and md5. These should generally be a part of our
releases.

```
/ # ls -l /
total 257984
...
-rw-r--r--    1 root     root     264106108 Oct  9 06:58 dhis.war
-rw-r--r--    1 root     root            44 Oct  9 06:58 md5sum.txt
-rw-r--r--    1 root     root            76 Oct  9 06:58 sha256sum.txt
...

/ # sha256sum -c /sha256sum.txt
/dhis.war: OK

/ # md5sum -c /md5sum.txt
/dhis.war: OK
```

