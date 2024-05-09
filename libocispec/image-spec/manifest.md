# OCI Image Manifest Specification

There are three main goals of the Image Manifest Specification.
The first goal is content-addressable images, by supporting an image model where the image's configuration can be hashed to generate a unique ID for the image and its components.
The second goal is to allow multi-architecture images, through a "fat manifest" which references image manifests for platform-specific versions of an image.
In OCI, this is codified in an [image index](image-index.md).
The third goal is to be [translatable](conversion.md) to the [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec).

This section defines the `application/vnd.oci.image.manifest.v1+json` [media type](media-types.md).
For the media type(s) that this is compatible with see the [matrix](media-types.md#compatibility-matrix).

# Image Manifest

Unlike the [image index](image-index.md), which contains information about a set of images that can span a variety of architectures and operating systems, an image manifest provides a configuration and set of layers for a single container image for a specific architecture and operating system.

## *Image Manifest* Property Descriptions

- **`schemaVersion`** *int*

  This REQUIRED property specifies the image manifest schema version.
  For this version of the specification, this MUST be `2` to ensure backward compatibility with older versions of Docker. The value of this field will not change. This field MAY be removed in a future version of the specification.

- **`mediaType`** *string*

  This property SHOULD be used and [remain compatible](media-types.md#compatibility-matrix) with earlier versions of this specification and with other similar external formats.
  When used, this field MUST contain the media type `application/vnd.oci.image.manifest.v1+json`.
  This field usage differs from the [descriptor](descriptor.md#properties) use of `mediaType`.

- **`artifactType`** *string*

  This OPTIONAL property contains the type of an artifact when the manifest is used for an artifact.
  This MUST be set when `config.mediaType` is set to the [scratch value](#example-of-a-scratch-config-or-layer-descriptor).
  If defined, the value MUST comply with [RFC 6838][rfc6838], including the [naming requirements in its section 4.2][rfc6838-s4.2], and MAY be registered with [IANA][iana].

- **`config`** *[descriptor](descriptor.md)*

    This REQUIRED property references a configuration object for a container, by digest.
    Beyond the [descriptor requirements](descriptor.md#properties), the value has the following additional restrictions:

    - **`mediaType`** *string*

        This [descriptor property](descriptor.md#properties) has additional restrictions for `config`.
        Implementations MUST support at least the following media types:

        - [`application/vnd.oci.image.config.v1+json`](config.md)

        Manifests for container images concerned with portability SHOULD use one of the above media types.
        Manifests for artifacts concerned with portability SHOULD use `config.mediaType` as described in [Guidelines for Artifact Usage](#guidelines-for-artifact-usage).

        If the manifest uses a different media type than the above, it MUST comply with [RFC 6838][rfc6838], including the [naming requirements in its section 4.2][rfc6838-s4.2], and MAY be registered with [IANA][iana].

    To set an effectively NULL or SCRATCH config and maintain portability the following is considered GUIDANCE.
    While an empty blob (`size` of 0) may be preferable, practice has shown that not to be ubiquitiously supported.
    Instead, the blob payload can be the most minimal content that is still valid JSON object: `{}` (`size` of 2).
    The blob digest of `{}` is `sha256:44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a`.
    See the [example SCRATCH config](#example-of-a-scratch-config-or-layer-descriptor) below, and `ScratchDescriptor` of the reference code.

- **`layers`** *array of objects*

    Each item in the array MUST be a [descriptor](descriptor.md).
    For portability, `layers` SHOULD have at least one entry.

    When the `config.mediaType` is set to `application/vnd.oci.image.config.v1+json`, the following additional restrictions apply:

    - The array MUST have the base layer at index 0.
    - Subsequent layers MUST then follow in stack order (i.e. from `layers[0]` to `layers[len(layers)-1]`).
    - The final filesystem layout MUST match the result of [applying](layer.md#applying-changesets) the layers to an empty directory.
    - The [ownership, mode, and other attributes](layer.md#file-attributes) of the initial empty directory are unspecified.

    For broad portability, if a layer is required to be used, use the SCRATCH layer.
    See the [example SCRATCH layer](#example-of-a-scratch-config-or-layer-descriptor) below, and `ScratchDescriptor` of the reference code.

    Beyond the [descriptor requirements](descriptor.md#properties), the value has the following additional restrictions:

    - **`mediaType`** *string*

        This [descriptor property](descriptor.md#properties) has additional restrictions for `layers[]`.
        Implementations MUST support at least the following media types:

        - [`application/vnd.oci.image.layer.v1.tar`](layer.md)
        - [`application/vnd.oci.image.layer.v1.tar+gzip`](layer.md#gzip-media-types)
        - [`application/vnd.oci.image.layer.nondistributable.v1.tar`](layer.md#non-distributable-layers)
        - [`application/vnd.oci.image.layer.nondistributable.v1.tar+gzip`](layer.md#gzip-media-types)

        Manifests concerned with portability SHOULD use one of the above media types.
        An encountered `mediaType` that is unknown to the implementation MUST be ignored.

        Entries in this field will frequently use the `+gzip` types.

        If the manifest uses a different media type than the above, it MUST comply with [RFC 6838][rfc6838], including the [naming requirements in its section 4.2][rfc6838-s4.2], and MAY be registered with [IANA][iana].

        See [Guidelines for Artifact Usage](#guidelines-for-artifact-usage) for other uses of the `layers`.

- **`subject`** *[descriptor](descriptor.md)*

    This OPTIONAL property specifies a [descriptor](descriptor.md) of another manifest.
    This value, used by the [`referrers` API](https://github.com/opencontainers/distribution-spec/blob/main/spec.md#listing-referrers), indicates a relationship to the specified manifest.

- **`annotations`** *string-string map*

    This OPTIONAL property contains arbitrary metadata for the image manifest.
    This OPTIONAL property MUST use the [annotation rules](annotations.md#rules).

    See [Pre-Defined Annotation Keys](annotations.md#pre-defined-annotation-keys).

## Example Image Manifest

*Example showing an image manifest:*
```json,title=Manifest&mediatype=application/vnd.oci.image.manifest.v1%2Bjson
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "size": 7023,
    "digest": "sha256:b5b2b2c507a0944348e0303114d8d93aaaa081732b86451d9bce1f432a537bc7"
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 32654,
      "digest": "sha256:9834876dcfb05cb167a5c24953eba58c4ac89b1adf57f28f2f9d09af107ee8f0"
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 16724,
      "digest": "sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b"
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 73109,
      "digest": "sha256:ec4b8955958665577945c89419d1af06b5f7636b4ac3da7f12184802ad867736"
    }
  ],
  "subject": {
    "mediaType": "application/vnd.oci.image.manifest.v1+json",
    "size": 7682,
    "digest": "sha256:5b0bcabd1ed22e9fb1310cf6c2dec7cdef19f0ad69efa1f392e94a4333501270"
  },
  "annotations": {
    "com.example.key1": "value1",
    "com.example.key2": "value2"
  }
}
```

## Example of a SCRATCH config or layer descriptor

Notice that the `mediaType` is subject to the usage or context, while the digest is specifically defined as `ScratchDigestSHA256`.
When the `ScratchDigestSHA256` is used, the media type SHOULD be set to `application/vnd.oci.scratch.v1+json` to differentiate the descriptor from one pointing to content.

```json,title=SCRATCH%20config&mediatype=application/vnd.oci.descriptor.v1%2Bjson
{
  "mediaType": "application/vnd.oci.scratch.v1+json",
  "size": 2,
  "digest": "sha256:44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a"
}
```

The content of the scratch blob is `{}` (`size` of 2).

## Guidelines for Artifact Usage

Content other than OCI container images MAY be packaged using the image manifest.
When this is done, the `config.mediaType` value MUST be set to a value specific to the artifact type or the [scratch value](#example-of-a-scratch-config-or-layer-descriptor).
If the `config.mediaType` is set to the scratch value, the `artifactType` MUST be defined.
If the artifact does not need layers, a single layer SHOULD be included with a non-zero size.
The suggested content for an unused layer is the [SCRATCH](#example-of-a-scratch-config-or-layer-descriptor) descriptor.

Here is an example manifest for a typical artifact:

```json,title=Artifact%20with%20config&mediatype=application/vnd.oci.image.manifest.v1%2Bjson
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.example.config.v1+json",
    "digest": "sha256:5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03",
    "size": 123
  },
  "layers": [
    {
      "mediaType": "application/vnd.example.data.v1.tar+gzip",
      "digest": "sha256:e258d248fda94c63753607f7c4494ee0fcbe92f1a76bfdac795c9d84101eb317",
      "size": 1234
    }
  ]
}
```

Here is an example manifest for a simple artifact without content in the config, using the scratch descriptor:

```json,title=Artifact%20without%20config&mediatype=application/vnd.oci.image.manifest.v1%2Bjson
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "artifactType": "application/vnd.example+type",
  "config": {
    "mediaType": "application/vnd.oci.scratch.v1+json",
    "digest": "sha256:44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a",
    "size": 2
  },
  "layers": [
    {
      "mediaType": "application/vnd.example+type",
      "digest": "sha256:e258d248fda94c63753607f7c4494ee0fcbe92f1a76bfdac795c9d84101eb317",
      "size": 1234
    }
  ]
}
```

Here is an example manifest for an artifact with only annotations set, and the content of both blobs set to the scratch descriptor:

```json,title=Minimal%20artifact&mediatype=application/vnd.oci.image.manifest.v1%2Bjson
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "artifactType": "application/vnd.example+type",
  "config": {
    "mediaType": "application/vnd.oci.scratch.v1+json",
    "digest": "sha256:44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a",
    "size": 2
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.scratch.v1+json",
      "digest": "sha256:44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a",
      "size": 2
    }
  ],
  "annotations": {
    "oci.opencontainers.image.created": "2023-01-02T03:04:05Z",
    "com.example.data": "payload"
  }
}
```

[iana]:         https://www.iana.org/assignments/media-types/media-types.xhtml
[rfc6838]:      https://tools.ietf.org/html/rfc6838
[rfc6838-s4.2]: https://tools.ietf.org/html/rfc6838#section-4.2
