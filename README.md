# Introduction

Building a connector locally:

This command will build the image ghcr.io/estuary/source-intercom:local (replace `source-intercom` with the connector that you want to build).
```
./local-build.sh source-intercom
```

You can then use `flowctl-go` commands to inspect the connector
e.g. to check the spec

```
flowctl-go api spec --image ghcr.io/estuary/source-intercom:local | jq
```

You can now modify patches, etc. and then re-run the commands
to build and check the spec

```
./local-build.sh source-intercom
flowctl-go api spec --image ghcr.io/estuary/source-intercom:local | jq
```

To test the connector in your local flow setup, you can push the connector with
a personal tag. First you need to login docker to `ghcr.io`, to do this you need
to create a [github personal
access
token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
with access to push to our container registry. Once you have this token, you can
login using the command below:

```
echo '<YOUR-TOKEN>' | docker login ghcr.io -u <YOUR-GITHUB-USERNAME> --password-stdin
```

Then you can push the connector by passing `-p` to `local.build.sh`. This
command will push the docker container with a tag unique to your codespace which will
be printed at the end.

```
./local-build.sh -p source-intercom
```

## Adding a new connector

1. First, find the latest tag of a connector. To do so, find your connector in airbyte repository [here](https://github.com/airbytehq/airbyte/blob/master/airbyte-integrations/connectors) and look at the `Dockerfile` of the connector, there will be a `io.airbyte.version` LABEL at the bottom of the `Dockerfile`, use the value of this label exactly as it is.

2. Then, run `new-connector.sh` script with two arguments: the connector name and the version tag. Example: `./new-connector.sh freshdesk 0.1.0`. This will create a new directory `airbyte-integrations/connectors/source-freshdesk`.

3. If you intend to patch the connector, these files can be placed in the root directory of the connector
and copied in Dockerfile. The following files are supported:
    1. `spec.patch.json`: to patch the connector's endpoint_spec, the patch is applied as specified by [RFC7396 JSON Merge](https://www.rfc-editor.org/rfc/rfc7396.txt)
    2. `spec.map.json`: to map fields from endpoint_spec. Keys and values are JSON pointers. Each key: value in this file is processed by moving whatever is at the value pointer to the key pointer
    3. `oauth2.patch.json`: to patch the connector's oauth2 spec. This patch overrides the connector's oauth2 spec
    4. `documentation_url.patch.json`: to patch the connector's
       documentation_url. Expects a single key `documentation_url` with a string value
    5. `streams/<stream-name>.patch.json`: to patch a specific stream's document schema
    6. `streams/<stream-name>.pk.json`: to patch a specific stream's primary key, expects an array of strings
    7. `streams/<stream-name>.normalize.json`: to apply data normalization functions to specific fields of documents generated by a capture stream. Normalizations are provided as a list (JSON array) of objects with keys `pointer` having a value of the pointer to the document field that the normalization will apply to, and `normalization` having a value of the name of the normalization function to apply to the field at that pointer. Normalization function names should provided as `snake_case`; see `airbyte_to_flow/src/interceptors/normalize.rs` for the supported normalization functions.

5. To build images for this new connector, you need to add this connector name
   to `.github/workflows/connectors.yml`,
   `jobs.build_connectors.strategy.matrix.connector` is the place to add the new
   connector.

Finally, make sure you build and run the connector once to make sure the spec is
valid.

## Updating an existing connector (no code)

To update a connector, open the `Dockerfile` of that connector, and in the `FROM airbyte/source-${NAME}:${VERSION}` line, replace `VERSION` with the latest tag of the connector.

## Updating an existing connector (connectors with code)

The `pull-connector.sh` script can update existing connectors as well. You can
just run:

```
./pull-connector.sh source-hubspot
```

The script will take you through a diff of the latest version from airbyte and
our local version, and will ask you about each file whether we should keep the
local file or take the file from upstream. A rule of thumb is that we want to
pull in code changes, but we usually keep our local version of `Dockerfile` and
`.dockerignore`. Don't forget to bump the version in `Dockerfile` if the changes
we are pulling from upstream are backward-incompatible.

## airbyte-to-flow

See the README file in `airbyte-to-flow` directory.
