# Introduction

Building a connector locally:

This command will build the image ghcr.io/estuary/source-intercom:local
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

To check the discovered bindings of a connector, you can use
`flowctl-go discover`

```
flowctl-go discover --image ghcr.io/estuary/source-intercom:local
```

Fill in the config file at `acmeCo/source-intercom.config.yaml`
and run discover again

```
flowctl-go discover --image ghcr.io/estuary/source-intercom:local
```

You can now check the discovered bindings in `acmeCo` and make sure that
the discovered bindings match your expectations

## Pulling a new connector

Use the `pull-connector.sh` script to pull a new connector (or update an
existing one):

```
./pull-connector.sh source-freshdesk
```

Once the connector is pulled in, you need to go through a few steps to make it
compatible with flow:

1. You need to make sure the Dockerfile of this connector uses airbyte-to-flow,
   see other connectors as an example, but essentially you need to:
    1. Update the `io.airbyte.version` to be in the format of v1
    2. Update the `ENTRYPOINT` and add a line before it like so:
      ```
      COPY --from=ghcr.io/estuary/airbyte-to-flow:dev /airbyte-to-flow ./
      ENTRYPOINT ["/airbyte/integration_code/airbyte-to-flow", "--connector-entrypoint", "python /airbyte/integration_code/main.py"]
      ```
    3. Add these two lines to the end of the Dockerfile:
      ```
      LABEL FLOW_RUNTIME_PROTOCOL=capture
      LABEL CONNECTOR_PROTOCOL=flow-capture
      ```

2. If you intend to patch the connector, these files can be placed in the root directory of the connector
and copied in Dockerfile. The following files are supported:
    1. `spec.patch.json`: to patch the connector's endpoint_spec, the patch is applied per RFC7396 JSON Merge
    2. `spec.map.json`: to map fields from endpoint_spec. Keys and values are JSON pointers. Each key: value in this file is processed by moving whatever is at the value pointer to the key pointer
    3. `oauth2.patch.json`: to patch the connector's oauth2 spec. This patch overrides the connector's oauth2 spec
    4. `documentation_url.patch.json`: to patch the connector's
       documentation_url. Expects a single key `documentation_url` with a string value
    5. `streams/<stream-name>.patch.json`: to patch a specific stream's document schema
    6. `streams/<stream-name>.pk.json`: to patch a specific stream's primary key, expects an array of strings

3. To add these patches to the Dockerfile, use the snippets below:
```
COPY documentation_url.patch.json ./
COPY spec.patch.json ./
COPY spec.map.json ./
COPY oauth2.patch.json ./
COPY streams/* ./streams/
```

4. Also make sure these files are not ignored by docker by adding the lines
   below to `.dockerignore`:
```
!*.patch.json
!*.map.json
!streams
```

5. To build images for this new connector, you need to add this connector name
   to `.github/workflows/connectors.yml`,
   `jobs.build_connectors.strategy.matrix.connector` is the place to add the new
   connector.

## airbyte-to-flow

See the README file in `airbyte-to-flow` directory.
