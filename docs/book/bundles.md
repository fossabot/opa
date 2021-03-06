# Bundles

OPA can periodically download bundles of policy and data from remote HTTP
servers. The policies and data are loaded on the fly without requiring a
restart of OPA. Once the policies and data have been loaded, they are enforced
immediately. Policies and data loaded from bundles are accessible via the
standard OPA [REST API](rest-api.md).

Bundles provide an alternative to pushing policies into OPA via the REST APIs.
By configuring OPA to download bundles from a remote HTTP server, you can
ensure that OPA has an up-to-date copy of policies and data required for
enforcement at all times.

## Bundle Configuration

You can configure OPA to periodically download policies by starting OPA with a
configuration file (both YAML and JSON files are supported):

```bash
opa run --server --config config.yaml
```

**config.yaml**:

```yaml
services:
  - name: acmecorp
    url: https://example.com/
    credentials:
      bearer:
        token: "Bearer <base64 encoded string>"
bundles:
  - name: http/example/authz
    service: acmecorp
    polling:
      min_delay_seconds: 60
      max_delay_seconds: 120
```

With this configuration OPA will attempt to download a bundle named
`http/example/authz` from `https://example.com/` every 1-2 minutes. The polling
delay is randomized (between the min and max) to stagger download requests when
there are a large number of OPAs requesting bundles.

### Configuration Format

```yaml
services:
  - name: string
    url: string
    headers: object
    credentials:
      bearer:
        token: string
bundles:
  - name: string
    service: string
    polling:
      min_delay_seconds: number
      max_delay_seconds: number
```

Most fields in the configuration are optional, however, to enable bundle
downloading, you must set the following fields:

| Field | Description |
| --- | --- |
| `services[_].name` | Unique name for the service. Referred to by plugins. |
| `services[_].url` | Base URL to contact the service with. |
| `bundles[_].name` | Name of the bundle to download. |
| `bundles[_].service` | Name of service to use to contact remote server. |

## Bundle Service API

OPA expects the service to expose an API endpoint that serves bundles. The
bundle API should allow clients to download named bundles.

```http
GET /bundles/<name> HTTP/1.1
```

The bundle name is hierarchical and contains slashes. For example, with the
configuration above, the client would execute an HTTP `GET` request against
`https://example.com/bundles/http/example/authz`.

If the bundle exists, the server should respond with an HTTP 200 OK status
followed by a gzipped tarball in the message body.

```http
HTTP/1.1 200 OK
Content-Type: application/gzip
```

OPA currently supports Bearer token authentication for external services.

## Bundle File Format

Bundle files are gzipped tarballs that contain policies and data. The data
files in the bundle must be organized hierarchically into directories inside
the tarball.

> The hierarchical organization indicates to OPA where to load the data files
> into the [the `data` Document](how-does-opa-work.md#the-data-document).

You can list the content of a bundle with `tar`.

```bash
$ tar tzf bundle.tar.gz
roles
roles/bindings
roles/bindings/data.json
roles/permissions
roles/permissions/data.json
http
http/example
http/example/authz
http/example/authz/authz.rego
```

In this example, the bundle contains one policy file (`authz.rego`) and two
data files (`bindings/data.json` and `permissions/data.json`).