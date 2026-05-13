# Design of an IVOA ConeSearch Service for the Rubin Science Platform

```{abstract}
The IVOA ConeSearch specification defines a simple protocol for querying astronomical source catalogs by sky position and radius.
This technote describes the design and implementation of a ConeSearch service for the Rubin Science Platform (RSP), covering backend query execution via TAP, streaming VOTable responses, per-collection configuration, VERB-driven column selection, and compliance with the ConeSearch 1.1 specification.
Key design decisions around column verbosity levels, TAP query execution, and multi-collection support are discussed with options and recommendations.
The service is built using the Safir FastAPI framework and deployed via Phalanx.
```

## 1. Introduction

The IVOA ConeSearch protocol provides a simple API for 
querying source catalogs, where a client requests a sky position and search 
radius and receives a VOTable listing all catalog entries within the cone.


ConeSearch is a widely adopted specification and is built into 
standard VO client libraries, including PyVO.

The RSP currently provides cone search-like functionality via the `datalinker` 
service, which exposes a `/api/datalink/cone_search` endpoint.
While this endpoint does provide a conesearch-like capability, it is not an IVOA-compliant ConeSearch implementation.

It constructs an ADQL query from client-supplied column names and issues a 
307 redirect to the TAP sync endpoint.

This approach differs from the ConeSearch standard, which 
requires the  service itself to return a VOTable with specific required 
fields and UCDs.

This technote describes the design of a dedicated IVOA-compliant ConeSearch service for the RSP.

### Goals

- Provide an IVOA ConeSearch 1.1 compliant service for RSP catalog data
- Support multiple named catalog collections (dp1, dp2, etc.) with independent configuration
- Return properly formed VOTable responses with required UCDs
- Support VERB-driven verbosity levels
- Authentication via Gafaelfawr
- Follow established SQuaRE patterns
- Expose VOSI capabilities and availability endpoints for VO client 
  discoverability (Not strictly required by the ConeSearch spec, but 
  commmon practice in the IVOA ecosystem)

### Out of Scope

- ConeSearch doesn't run queries, instead all queries are delegated to TAP
- ConeSearch is a synchronous protocol, so an async interface for it is out of 
  scope
- No support for POST requests (the ConeSearch spec is GET-only)

## 2. ConeSearch Standard Overview

The IVOA ConeSearch 1.1 specification defines a simple HTTP GET interface.

### Query Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `RA` | Yes | Right ascension of search position (ICRS decimal degrees, 0-360) |
| `DEC` | Yes | Declination of search position (ICRS decimal degrees, -90 to +90) |
| `SR` | Yes | Search radius (decimal degrees, 0 to MaxSR) |
| `VERB` | No | Verbosity level: 1 (minimal), 2 (default), 3 (all columns) |
| `TIME` | No | Temporal filter as MJD value or range (e.g. `58000` or `58000/59000`) |
| `MAXREC` | No | Maximum number of records to return |
| `RESPONSEFORMAT` | No | Response format (default: `application/x-votable+xml`) |

### Response Format

Responses must be VOTable XML with a single `RESOURCE` containing one `TABLE`.
The following three `FIELD` elements are mandatory:

| UCD | Data Type | Description |
|-----|-----------|-------------|
| `ID_MAIN` | char | Unique source identifier |
| `POS_EQ_RA_MAIN` | double | Right ascension |
| `POS_EQ_DEC_MAIN` | double | Declination |

The VERB parameter controls the column set returned:

- **VERB=1**: The three required fields only
- **VERB=2**: A useful intermediate set (default)
- **VERB=3**: All available columns

### Error Responses

Errors must be returned as VOTable XML containing `INFO` or `PARAM` elements with the error message.
A plain HTTP error response is not compliant.

### VOSI Endpoints

ConeSearch 1.1 requires a `/capabilities` endpoint and encourages `/availability`.
TAP, SIA, and other RSP services implement both, and VO client libraries such as PyVO use `/capabilities` to discover what a service supports.
This service implements both endpoints.

## 3. Existing RSP Context

### datalinker cone_search

The `datalinker` service provides a `/api/datalink/cone_search` endpoint which provides similar functionality, but not in a way that is complient to the IVOA ConeSearch standard.
It accepts `table`, `ra_col`, `dec_col`, `ra_val`, `dec_val`, and `radius` as parameters, constructs an ADQL `CONTAINS/CIRCLE` query and issues a 307 redirect to `/api/tap/sync`.

The new ConeSearch service replaces this pattern with a server-owned, 
per-collection configuration that maps catalog columns to the required 
ConeSearch columns, executes the TAP query internally and returns a VOTable 
directly to the client in accordance to the specification.

### SIA Service

The SIA v2 service (`sia`) is the closest analogue service.
It supports multiple named data collections routed as `/api/sia/{collection}/query`, implements VOSI endpoints, returns VOTable responses via `astropy` and follows the same Safir/Phalanx patterns.
The ConeSearch service will follow the same multi-collection routing structure and VOSI patterns.

### TAP Service

All catalog queries are delegated to the relevant RSP TAP service.
Each collection is configured with its own TAP endpoint, allowing different data releases to be served by different TAP instances.
The choice of TAP execution mode (synchronous vs asynchronous UWS) is a key design decision discussed in this document.


## 4. Requirements

- Accept `RA`, `DEC`, `SR` query parameters per the ConeSearch 1.1 specification
- Accept optional `VERB` (1, 2, 3), `TIME`, `MAXREC`, and `RESPONSEFORMAT` parameters
- Validate parameter ranges: RA in [0, 360], DEC in [-90, 90], SR in [0, MaxSR]
- Return ConeSearch-compliant VOTable with the three required fields
- Return VOTable error responses for invalid parameters or TAP errors
- Support multiple named collections with independent table, column, and MaxSR configuration
- Enforce a configurable MaxSR per collection (default: 180 degrees)
- Enforce a configurable MaxRecords per collection, limiting the number of rows returned
- Authenticate requests via Gafaelfawr delegated token (requiring `read:tap` scope), forwarded to TAP
- Expose `/capabilities` (required) and `/availability` (encouraged) VOSI endpoints per collection
- Return `Content-Type: application/x-votable+xml`
- Handle case-insensitive query parameters

## 5. Architecture

The service is a standard Safir FastAPI application following established SQuaRE patterns.
It exposes per-collection endpoints under `/api/conesearch/{collection}/`, where each collection corresponds to a named catalog (e.g. a data release or catalog type).

### Request Flow

1. Client sends `GET /api/conesearch/{collection}/query?RA=…&DEC=…&SR=…&VERB=…`
2. Handler validates parameters (range checks, MaxSR limit)
3. Handler resolves the columns for the requested VERB level based on the collection configuration
4. Handler constructs an ADQL `CONTAINS/CIRCLE` query against the configured table
5. Service opens a streaming request to the TAP `/sync` endpoint, forwarding the Gafaelfawr delegated token
6. The first chunk of the TAP response is buffered to detect `QUERY_STATUS=ERROR` before committing to a success response
7. On success, the TAP VOTable byte stream is forwarded directly to the client via `StreamingResponse`

Error cases at steps 2 and 5-6 produce VOTable error responses (not HTTP errors).


### Key Components

**`Config`** Config holds service-wide settingsand a dictionary of named collections (collections: dict[str, CollectionConfig]). 
Each CollectionConfig specifies the collection's TAP endpoint (tapUrl), the fully qualified catalog table name, the column names that map to the three required ConeSearch roles (idColumn, raColumn, decColumn), the maxSr and maxRecords limits and the path to the VERB column metadata file (verbColumnsPath).

**`Factory`** is a per-request object that holds the HTTP client, delegated token, logger, and event publisher, and exposes a `create_conesearch_service()` method. It follows the same pattern used in other phalanx services.

**`ConeSearchService`** executes the TAP query and streams the VOTable response back to the caller as an async generator. It is instantiated per request via the `Factory`.


### Streaming Response

The TAP VOTable is streamed directly to the client without being loaded into memory.
The service buffers only the first 8 KB of the TAP response to detect a `QUERY_STATUS=ERROR` VOTable (error documents are small and always appear in the XML header). Once the response is confirmed to be a success, the buffered bytes and all subsequent chunks are forwarded as a `StreamingResponse`.

This avoids loading potentially multi-GB result sets into process memory on the server.
Note that VO client libraries such as PyVO still buffer the full response client-side to parse the VOTable.

UCD rewriting (applying `ID_MAIN`, `POS_EQ_RA_MAIN`, `POS_EQ_DEC_MAIN` to the configured columns) is left to the TAP service.


## 6. API

Similar to the SIAv2 service, all endpoints will be available at the `/api/conesearch/{collection}/` prefix.
The `{collection}` path parameter identifies the named catalog collection.

### `GET /api/conesearch/{collection}/query`

Executes a cone search query against the named collection.

**Query Parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `RA` | Yes | Right ascension (decimal degrees) |
| `DEC` | Yes | Declination (decimal degrees) |
| `SR` | Yes | Search radius (decimal degrees) |
| `VERB` | No | Verbosity level: 1, 2, or 3 (default: 2) |
| `TIME` | No | Temporal filter as MJD value or range (e.g. `58000/59000`) |
| `MAXREC` | No | Maximum number of records to return |
| `RESPONSEFORMAT` | No | Response format (default: `application/x-votable+xml`) |

**Responses:**

- `200 OK` - VOTable XML (`text/xml`) containing matching sources
- `200 OK` - VOTable XML with `INFO` error element for invalid parameters or TAP errors (per spec)
- `401 Unauthorized` - missing or invalid Gafaelfawr token (requires `read:tap` scope)

### `GET /api/conesearch/{collection}/capabilities`

Returns a VOSI capabilities document describing the capabilities of the 
service. Specifically, it should outline the ConeSearch interface and its parameters.

### `GET /api/conesearch/{collection}/availability`

Returns a VOSI availability document indicating whether the service is up and running.

### `GET /api/conesearch/`

Returns application metadata.

## 7. Design Options

### 7.1 VERB Column Selection

The ConeSearch VERB parameter controls how many columns are returned.
VERB=1 is the minimum (three required fields), VERB=2 is the default intermediate set, and VERB=3 returns all columns via `SELECT *`.

Column lists for VERB=1 and VERB=2 are defined inline in the collection configuration, following the established pattern where application config lives in `values.yaml` and is rendered into a single ConfigMap.
This avoids the need for separate per-collection files and keeps all configuration in one place.

```yaml
# values.yaml
config:
  collections:
    dp02:
      verb1Columns: [objectId, coord_ra, coord_dec]
      verb2Columns: [objectId, coord_ra, coord_dec, r_calibFlux, g_calibFlux,
                     i_calibFlux, r_cModelFlux, detect_isPrimary, refExtendedness]
```

If `verb1Columns` or `verb2Columns` is empty or not set for a collection, the service falls back to `SELECT *` for that level, equivalent to VERB=3.
Column selection is applied in the ADQL query via `SELECT TOP {maxRecords} col1, col2, ...`.

### 7.2 TAP Query Execution: Synchronous vs Asynchronous

ConeSearch is a synchronous protocol from the client's point of view. The 
client sends a GET request and expects a VOTable response in the same HTTP connection.
Internally there are two ways to execute the underlying TAP query.

**Option A - TAP sync endpoint + `httpx.AsyncClient`**

The service calls the TAP `/sync` endpoint using `httpx.AsyncClient` and `await`s the response.
The FastAPI event loop remains free while waiting, so no thread pool is needed.
This is the pattern that is used in `datalinker`.

The TAP sync timeout in `lsst-tap-service` currently is configured to 60 
seconds.
A timeout returns HTTP 200 with a VOTable error response rather than a 503, which means the service can forward the TAP error response directly to the client in the correct ConeSearch format.

**Option B - TAP async UWS endpoint with inline polling**

The other option is to use asynchronous TAP queries. With this approach, the 
service submits a UWS job to the TAP `/async` endpoint, polls the job 
status until completion, retrieves the result VOTable, deletes the job, and 
returns the result to the client. 
If using pyvo this workflow can be hidden if using the `run_async` method, or it can be implemented natively with `httpx` calls to the TAP async endpoints.
If we poll manually, we can use `httpx.AsyncClient` with 
`asyncio.sleep`between attempts to avoid blocking the event loop.
This bypasses the TAP sync timeout entirely and can handle arbitrarily long queries.

The PyVO `run_async` helper is synchronous, so it would require `asyncio.
to_thread`.

A native `httpx`-based polling loop keeps the event loop free but adds implementation complexity.


**Summary**

Both options keep the event loop non-blocking.
The choice mainly depends on whether 60-second TAP sync timeout is an 
acceptable constraint.

With `max_records` bounding result size, anecdotally most ConeSearch queries 
over reasonable search radius will complete within this limit.
Option B should be preferred only if experience shows queries 
regularly approaching or exceeding the timeout. 

**Option A was selected.** The TAP response is streamed via `httpx.AsyncClient.stream()`, keeping the server event loop non-blocking and avoiding in-process buffering of large results. Option B remains available if the 60-second sync timeout proves limiting in practice.

### 7.3 UCD Mapping

The ConeSearch spec requires specific UCDs on the three mandatory fields.
These old-style UCDs are not present in our existing catalogs which instead uses UCD1+.
This implementation adds a custom UCD mapping as a query parameter to the TAP service which requests the UCD version for the relative columns, and then streams the TAP VOTable directly without transformation,  so the TAP-assigned UCDs are passed through unchanged.

## 8. Configuration

As is the pattern for other RSP applications, the service will be 
configured via a YAML file:

```yaml
pathPrefix: /api/conesearch
logLevel: INFO
profile: production

collections:
  dp02:
    tapUrl: https://data.lsst.cloud/api/tap
    table: dp02_dc2.Object
    idColumn: objectId
    raColumn: coord_ra
    decColumn: coord_dec
    maxSr: 180.0
    maxRecords: 10000
    verb1Columns: [objectId, coord_ra, coord_dec]
    verb2Columns: [objectId, coord_ra, coord_dec, r_calibFlux, g_calibFlux,
                   i_calibFlux, r_cModelFlux, detect_isPrimary, refExtendedness]
  dp1:
    tapUrl: https://data.lsst.cloud/api/tap
    table: dp1.Object
    idColumn: objectId
    raColumn: z_ra
    decColumn: z_dec
    maxSr: 180.0
    maxRecords: 10000
    verb1Columns: [objectId, z_ra, z_dec]
    verb2Columns: [objectId, z_ra, z_dec, r_calibFlux, g_calibFlux,
                   i_calibFlux, r_cModelFlux, detect_isPrimary, refExtendedness]
```

Each collection specifies its own `tapUrl`, allowing different collections to be served by different TAP instances.

### Data Discovery for TAP URLs

Worth noting that with Repertoire providing service/data discovery, ConeSearch could query Repertoire at startup to resolve the TAP URL for each configured collection, 
rather than requiring hardcoded `tapUrl` in every deployment's configuration.

However, before adopting we need to verify he mapping from a ConeSearch 
collection name (e.g.`dp02`) to a Repertoire dataset and then to a specific 
TAP URL follow a clear convention.
The initial implementation will use explicit per-collection `tapUrl` configuration for simplicity.
Once an initial MVP service is developed, Repertoire-based discovery is a 
natural follow-up change.

### Position Column Variation Across Data Releases

The column names for sky position may not be uniform across data releases.
In dp02, the Object table uses `coord_ra` and `coord_dec` whereas dp1 uses band-specific position columns (`z_ra`, `z_dec`).
This is handled naturally by the per-collection `raColumn` and `decColumn` configuration fields.


## 9. Phalanx Deployment

The service will be deployed as a Phalanx application at `applications/conesearch/`.
Per-environment configuration (per-collection TAP URLs, table names, Gafaelfawr scopes) will be managed through Phalanx values files following the established pattern used by `sia` and `datalinker`.

## References

- IVOA ConeSearch 1.1: https://www.ivoa.net/documents/ConeSearch/20200828/WD-ConeSearch-1.1-20200828.html
- IVOA VOSI: https://www.ivoa.net/documents/VOSI/
- datalinker: https://github.com/lsst-sqre/datalinker
- SIA service: https://github.com/lsst-sqre/sia
- Safir: https://safir.lsst.io/
- Gafaelfawr: https://gafaelfawr.lsst.io/
- PyVO: https://pyvo.readthedocs.io/
- Phalanx: https://phalanx.lsst.io/
