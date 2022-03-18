# Proposal A

Add a new manifest, that provides a superset of capabilities of the image manifest, with a reduced set of constraints enabling the container runtime image artifact, and new artifact types that include reference types.

The medications include:

- [New artifact manifest](#artifact-manifest)
- [Descriptor (OPTIONAL) additional property](#descriptor-properties)
- [Distribution - Referrers API](#registry-http-api)

## Artifact Manifest

The `artifact.manifest` provides an optional collection of `blobs`, an optional `subject` reference to the manifest of another artifact and an `artifactType` to differentiate types of artifacts (such as signatures, sboms and security scan results).

## Links

| Description                                 | Link                        |
| ------------------------------------------- | --------------------------- |
| GitHub issue where this was first proposed:  | [OCI Artifact Manifest - with weak reference support #27](https://github.com/opencontainers/artifacts/pull/27)<br>[OCI Artifact Manifest, Phase 1-Reference Types #29](https://github.com/opencontainers/artifacts/pull/29) |
| Current ORAS Specifications:   |  [Manifest](https://github.com/oras-project/artifacts-spec/blob/main/artifact-manifest.md) and [Referrers API](https://github.com/oras-project/artifacts-spec/blob/main/manifest-referrers-api.md) | 
| Implementations | Registry: [CNCF DIstribution (fork for WIP)](https://github.com/oras-project/distribution/tree/ext_reftype)<br>Registry: [ZOT Project (merged)](https://github.com/project-zot/zot/issues/264)<br>Registry client: [oras-project/oras/tree/artifacts](https://github.com/oras-project/oras/tree/artifacts)<br>Azure Docs: [Push, pull, discover supply chain artifacts](https://aka.ms/acr/oras-artifacts) |

### Modifications 

The Artifact manifest is similar to the [OCI image manifest][oci-image-manifest-spec], but removes constraints defined on the image-manifest such as a required `config` object and required & ordinal `layers`.
It adds a `subject` property supporting a graph of independently linked artifacts.
The addition of a new manifest does not change, nor impact the `image.manifest`.
It provides a means to define a wide range of artifacts, including a chain of related artifacts enabling SBoMs, signatures and metadata that can be related to an `image.manifest`, `image.index` or another `artifact.manifest`.
By defining a new manifest, registries and clients opt-into new capabilities, without breaking existing registry and client behavior or setting expectations for scenarios to function when the client and/or registry may not yet implement new capabilities

### JSON Schema

```jsonc
{
  "mediaType": "application/vnd.oci.artifact.manifest.v1+json",
  "artifactType": "signature/example",
  "blobs": [
    {
      "mediaType": "application/tar",
      "digest": "sha256:9834876dcfb05cb167a5c24953eba58c4ac89b1adf57f28f2f9d09af107ee8f0",
      "size": 32654
    }
  ],
  "subject": {
    "mediaType": "application/vnd.oci.image.manifest.v1+json",
    "digest": "sha256:73c803930ea3ba1e54bc25c2bdc53edd0284c62ed651fe7b00369da519a3c333",
    "size": 16724
  },
  "annotations": {
    "com.example.signature.subject": "wabbit-networks.io"
  }
}
```
### Manifest Properties

- **`mediaType`** *string*

  This field contains the `mediaType` of this document, differentiating from [image-manifest][oci-image-manifest-spec] and [image-index][oci-image-index]. The `mediaType` for this manifest type MUST be `application/vnd.oci.artifact.manifest.v1+json`, where the version WILL change to reflect newer versions.

- **`artifactType`** *string*

  The REQUIRED `artifactType` is a unique value, as registered with [iana.org][registering-iana].
  The `artifactType` values are equivalent to the values used in the `manifest.config.mediaType` in [OCI Artifacts][oci-artifacts].
  Examples include `sbom/example`, `ice-cream/example`.
  For details on creating a unique `artifactType`, see [OCI Artifact Authors Guidance][oci-artifact-authors]

- **`blobs`** *array of objects*

  An OPTIONAL collection of 0 or more blobs. The blobs array is analogous to [oci.image.manifest layers][oci-image-manifest-spec-layers], however unlike [image-manifest][oci-image-manifest-spec], the ordering of blobs is specific to the artifact type. Some artifacts may choose an overlay of files, while other artifact types may store independent collections of files.
    - Each item in the array MUST be an [artifact descriptor][descriptor], and MUST NOT refer to another `manifest` providing dependency closure.
    - The max number of blobs is not defined, but MAY be limited by [distribution-spec][oci-distribution-spec] implementations.

- **`subject`** *descriptor*

   An OPTIONAL reference to any existing manifest within the repository. When specified, the artifact is said to be dependent upon the referenced `subject`.
   - The item MUST be an [artifact descriptor][descriptor] representing a manifest. Descriptors to blobs are not supported. The registry MUST return a `400` response code when `subject` is not found in the same repository, and not a manifest.

- **`annotations`** *string-string map*

    This OPTIONAL property contains arbitrary metadata for the artifact manifest.
    This OPTIONAL property MUST use the [annotation rules][annotations-rules].
    This map MAY contain some or all of the pre-defined keys listed below.

    **Pre-Defined Annotation Keys:**
    This defines a set of keys that have been pre-defined for use by authors of the artifact.
    - `org.oci.artifact.created` date and time on which the artifact was created (string, date-time as defined by [RFC 3339][rfc-3339])

### Descriptor Properties

Registries and clients work with descriptors as the means to establish discovery and links. To support hashing of different types, enabling filtering, the `artifactType` property is added as an optional property to the descriptor: 

- **`artifactType`** *string*

  This OPTIONAL property defines the type or Artifact, differentiating artifacts that use the `application/vnd.oci.artifact.manifest`.
  When the descriptor is used for blobs, this property MUST be empty.

## Registry HTTP API

The `referrers` extension API is provided to discover these artifacts.
An artifact client would parse the returned [artifact descriptors][descriptor], determining which  artifact manifest they will pull and process.

The `referrers` API returns all artifacts that have a `subject` of the given manifest digest.
Reference artifact requests are scoped to a repository, ensuring access rights for the repository can be used as authorization for the referenced artifacts.

```http
GET /v2/{repository}/_oci/ext/discover
```

The response SHOULD contain an extension with the name of `org.oci.referrers`
and the `url` path where the referrers can be requested.

```jsonc
200 OK
Content-Length: <length>
Content-Type: application/json

{
    "extensions": [
        {
            "name": "org.oci.referrers",
            "description": "Reference Type listing API",
            "url": "_reftype/artifacts/referrers"
        }
    ]
}
```

### API Path

The `referrers` api are provided on the [distribution-spec][oci-distribution-spec] paths as described below.
The path of the referrers api provides consistent namespace/repository paths, enabling registry operators to implement consistent auth access, using existing tokens for content.

**template:**

```rest
GET /v2/{repository}/_reftype/artifacts/referrers?digest={digest}
```

**expanded example:**

```rest
GET /v2/net-monitor/_reftype/artifacts/referrers?digest=sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b
```

### Artifact Referrers API results

- Implementations MUST implement [paging](#paging-results).
- Implementations MUST implement [sorting](#sorting-results)
- Implementations SHOULD implement [`artifactType` filtering](#filtering-results).

Some artifacts types including signatures, may return multiple signatures of the same `artifactType`.
For cases where multiple artifacts are returned to the client, it may be necessary to pull each artifact's manifest in order to determine whether or not the full artifact is needed.
Maintainers of the standards utilizing references SHOULD define standard sets of annotations that will allow clients to determine whether or not each artifact needs to be downloaded in full.

While this will cause additional round trips, manifests are typically small in comparison to the full pull time for a manifest and its blobs or layers.

This paged result MUST return the following elements:

- `referrers`: A list of [artifact descriptors][descriptor] that reference the
given manifest. The list MUST include these references even if the given
manifest does not exist in the repository. The list MUST be empty
if there are no artifacts referencing the given manifest.

**example result of artifacts that reference the `net-monitor` image:**

```json
{
  "referrers": [
    {
      "digest": "sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b",
      "mediaType": "application/vnd.oci.reftype.artifact.manifest.v1+json",
      "artifactType": "signature/example",
      "size": 312
    },
    {
      "digest": "sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b",
      "mediaType": "application/vnd.oci.reftype.artifact.manifest.v1+json",
      "artifactType": "sbom/example",
      "size": 237
    }
  ]
}
```

**example result for a manifest that has no artifacts referencing it:**

```json
{
  "referrers": []
}
```

### Paging Results

The `referrers` API MUST provide for paging, returning a list of [artifact descriptors](./descriptor.md).
Page size can be specified by adding a `n` parameter to the request URL, indicating that the response should be limited to `n` results.

- If specified, servers MAY return up to `n` items from the entire result set.
- When `n` is not provided, servers MAY return a default number of items, which may be implementation specific.

A paginated flow begins as:

**template:**

```rest
GET /v2/{repository}/_reftype/artifacts/referrers?digest={digest}&n=<integer>
```

**expanded example:**

```rest
GET /v2/{repository}/_reftype/artifacts/referrers?digest=sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b&n=10
```

The above specifies that a referrers response should be returned limiting the number of results to `n`.
The response to such a request would look as follows:

```json
200 OK
RefType-Api-Version:reftype/1.0
Link: <url>; rel="next"

{
  "referrers": [
    {
      "digest": "<string>",
      "mediaType": "<string>",
      "artifactType": "<string>",
      "size": <integer>
    },
    ...
  ]
}
```

The above includes up to `n` items from the result set. If there are more items, the URL for the next collection is
encoded in a [RFC5988][rfc5988] `Link` header, as a "next" relation. Clients SHOULD treat this as an opaque value and not try to
construct it themselves.

- The presence of the `Link` header communicates to the client that the server has more items. Clients are expected
  to follow the link to fetch the next page of items, irrespective of the number of items received in the current
  response.
- If the header is not present, clients can assume that all items have been received.

> NOTE: In the request template above, the brackets around the url are required.

For example, if the url is:

```
http://example.com/v2/hello-world/_reftype/artifacts/referrers?digest=sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b&n=5&nextToken=abc
```

The value of the header would be:

```
<http://example.com/v2/hello-world/_reftype/artifacts/referrers?digest=sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b&n=5&nextToken=abc>; rel="next"`.
```

Please see [RFC5988][rfc5988] for details.

### Sorting Results

The `/referrers` API MUST allow for artifacts to be sorted by the date and time in which they were created, which SHOULD be included in the artifact manifest's list of `annotations`.
The artifact's creation time MUST be the value of the `org.oci.artifact.created` annotation, as specified in the [artifact-manifest spec][artifact-manifest-spec].
The results of the `/referrers` API MUST list artifacts that were created more recently first.
Artifacts that do not have the `org.oci.artifact.created` annotation MUST appear after those with creation times specified in the list of results.
There is no specified ordering for artifacts that do not include the creation time in their list of `annotations`.

### Filtering Results

The `referrers` API MAY provide for filtering of `artifactTypes`.
Artifact clients MUST account for implementations that MAY NOT support filtering.
Artifact clients MUST revert to client side filtering to determine which `artifactTypes` they will process.

Request referenced artifacts by `artifactType`

**template:**

```rest
GET /v2/{repository}/_reftype/artifacts/referrers?digest={digest}&n=10&artifactType={artifactType}
```

**expanded example:**

```rest
GET /v2/net-monitor/_reftype/artifacts/referrers?digest=sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b&n=10&artifactType=signature%2Fexample
```

### Push Validation

Following the [distribution-spec push api](https://github.com/opencontainers/distribution-spec/blob/main/spec.md#push), all blobs *and* the `subject` descriptors SHOULD exist when pushed to a distribution instance.

### Lifecycle Management

Registries MAY treat the lifecycle of a reference type object, such as an SBoM or signature, as being tied to its `subject`. In such registries, when the `subject` is deleted or marked for garbage collection, the defined artifact is subject to deletion as well, unless the artifact is tagged.

## Further Reading

- [Usage and Scenarios](https://github.com/oras-project/artifacts-spec/blob/main/scenarios.md)
- [Comparing the ORAS Artifact Manifest and OCI Image Manifest][manifest-differences]

[annotations-rules]:               https://github.com/opencontainers/image-spec/blob/main/annotations.md#rules
[descriptor]:                      https://github.com/oras-project/artifacts-spec/blob/main/descriptor.md
[manifest-differences]:            https://github.com/oras-project/artifacts-spec#comparing-the-oras-artifact-manifest-and-oci-image-manifest
[oci-artifact-authors]:            https://github.com/opencontainers/artifacts/blob/master/artifact-authors.md
[oci-artifacts]:                   https://github.com/opencontainers/artifacts
[oci-distribution-spec]:           https://github.com/opencontainers/distribution-spec
[oci-image-index]:                 https://github.com/opencontainers/image-spec/blob/master/image-index.md
[oci-image-manifest-spec-layers]:  https://github.com/opencontainers/image-spec/blob/master/manifest.md#image-manifest-property-descriptions
[oci-image-manifest-spec]:         https://github.com/opencontainers/image-spec/blob/master/manifest.md
[registering-iana]:                https://github.com/opencontainers/artifacts/blob/master/artifact-authors.md#registering-unique-types-with-iana
[rfc-3339]:                        https://tools.ietf.org/html/rfc3339#section-5.6
[rfc5988]:                         https://datatracker.ietf.org/doc/html/rfc5988
[oras-azure]:                      https://aka.ms/acr/oras-artifacts