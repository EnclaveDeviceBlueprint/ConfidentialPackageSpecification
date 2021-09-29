# The Confidential Package File Format

## Introduction

This document is the formal specification for the Confidential Package File Format.

## Readership

This document is intended to be used by anyone who is developing or maintaining software tools that
need to either consume or produce Confidential Package files.

## Status
This document is a DRAFT and is subject to change without notice.

## Prerequisites

Readers should have some familiarity with [binary](https://en.wikipedia.org/wiki/Binary_file) file
formats, and be comfortable with low-level specifications of such formats down to the level of
individual byte fields and file offsets. Readers also need to be familiar with
[JSON](https://www.json.org) in order to understand the manifest specification. Since this document
is concerned with confidential (encrypted) content, a basic understanding of cryptographic concepts
and their typical applications is also desirable. In particular, it would be useful to understand
the principles of symmetric and asymmetric encryption, private keys, public keys, digital signatures
and certificates.

## Overview

A Confidential Package is a self-contained binary structure that is designed to store and/or
transport a confidential payload, typically a compiled executable object such as an application.
Confidential Packages are typically stored as files on disk, and this document specifies the format
of such files. Confidential Package files, when they reside on disk, conventionally have the
filename extension ".cpk", although this is only a convention, and is not a requirement of the
specification.

Confidential Package files are self-contained, meaning that they embed the confidential content
directly within the package, rather than reference it from another source. Although this
specification allows for plaintext embedding of such content, the expected practice would be for
such content to be encrypted.

A typical use case for a Confidential Package file is to store a compiled confidential application
that is intended to be deployed within a secure boundary such as a [Trusted Execution Environment
(TEE)](https://en.wikipedia.org/wiki/Trusted_execution_environment) on a target computing device.
Such an application would be built from source code using applicable tools, and then encrypted and
stored within a Confidential Package. The Confidential Package can then be stored or transmitted
through standard channels without any additional protection, due the encryption applied to the
payload. Decryption and execution then occurs within the TEE of the target computing device.

In addition to the confidential payload itself, the package is able to either embed or reference
additional resources such as signatures and certificates. The format also allows for the payload and
the package to be signed and certified by different entities, whereever this might be useful.

Confidential Packages deliberately avoid embedding any information about how to obtain the key for
decryption. This is a security defence mechanism and is by design. Distribution of keys for
decryption is the responsibility of the end-to-end environment in which the packages are being
deployed.

## Design Goals

The design aims for the Confidential Package file format are as follows:-

- Portability and agnosticism with respect to processor architectures, operating systems, TEE
   implementations, cloud services and storage/deployment models.
- Flexibility with respect to encryption key and certificate management.
- Supportive of flexible signing models and trust chains, allowing for independence of application
   vendors relative to the device firmware.
- Designed-in versioning and future extensibility.
- Agility with respect to cryptographic algorithm use.
- Simplest possible binary structure. Richly-structured data should present in a form that is
   readily processed by popular tools and libraries. Very little custom code should be needed to
   process the low-level binary aspects of the format.
- Scalable with respect to the embedded confidential payload size. It should be possible to both
   consume and produce Confidential Package files without requiring the whole content to be in
   system memory at any one time.
- Interoperable with any relevant existing standards, or at least not in competition with them.
- Efficiency in terms of size - a Confidential Package file should not be significantly larger than
   the confidential payload that it contains.

The remainder of this document presents the Confidential Package file format in detail.

## Structure

Confdential Package files are binary files. They are intended to be machine-readable rather than
human-readable.

The overall structure of a Confidential Package file is known as the *frame*, which is a very
minimal and simple binary skeleton that is designed to be very easy to navigate or produce. The
frame has a simple header, followed by a number of variable-length *streams*.

The streams are simply binary blobs of varying sizes, arramged contiguously within the file. One of
the streams will be the confidential payload itself - typically an encrypted executable file
intended for execution within a TEE. Other streams are used to embed additional binary or text data
objects that are needed to process the overall package.

Another one of the streams is assigned to be the *manifest*. The manifest is a JSON document, which
contains all of the information and metadata required to fully understand and process the package
contents. The manifest is a rich data structure, but because it is JSON it can be processed using
standard JSON tools and libraries. The JSON manifest structure is defined further down in this
document.

Manifest streams are small: typically just a few kilobytes in size. Other streams within the file,
especially the confidential payload, might be significantly larger. Each stream is referred to using
its integer *stream index*, which is a 16-bit numerical value based at zero. The JSON manifest uses
stream indexes to refer to streams and indicate their role in processing the package. Locating a
given stream within the file is very straightforward, because the frame header includes a *stream
table*, which provides the file offset at which each stream begins.

## Endianness

All multi-byte numerical values in the Confidential Package file format are represented in
*little-endian* order.

## Magic Number

The first four bytes of any Confidential Package file contain the hexadecimal "magic number" value
`0x0ECCF00D`. This is always present in little-endian byte order. Hence the first four bytes in
sequence are always `0x0D`, `0xF0`, `0xCC`, `0x0E`. These opening four bytes are the signature
sequence that indicate a Confidential Package file. A file that does not begin with these four bytes
cannot be a valid Confidential Package file.

## Frame Header

The four magic number bytes described above make up the first component of what is called the *frame
header*, which is found at the start of every Confidential Package file. The frame header is a
fixed, 16-byte structure, of which the first four bytes are the magic number. The fixed header
structure is given in the following table. All fields are represented in little-endian byte order,
regardless of the environment in which the Confidential Package file is being produced or consumed.
This is to ensure maximum portability and platform-neutrality.

| Field            | Size (in bytes) | Description                              |
|------------------|-----------------|------------------------------------------|
| Magic Number     | 4               | The fixed value `0x0ECCF00D`.            |
| Frame Version    | 2               | The only supported value is `1`.         |
| Flag             | 2               | Reserved. Must be `0`.                   |
| Stream Count     | 2               | The number of streams in the package.    |
| Manifest Index   | 2               | Zero-based index of the manifest stream. |
| Manifest Type    | 2               | The only supported value is `1`.         |
| Manifest Version | 2               | The only supported value is `1`.         |

Most of these fields are designed to support future evolution and extensibility for the file format.
The most important fields are the stream count and the manifest index. The stream count indicates
the number of streams within the file, and the manifest index allows the consumer to locate and
parse the package manifest, which is essential for processing the remainder of the package.

## Stream Table

Immediately following the frame header (and hence beginning at the 17th byte of the file) is the
*stream table*. The stream table is a sequence of N pairs of 64-bit numerical values, where N is the
number of streams as specified by the stream count in the frame header. Each stream table entry is a
pair of numbers: the stream *offset* and the stream *size*. Stream offsets describe the position of
the start of the stream within the file. However, these offsets are **not** relative to the
beginning of the file. They are actually offsets *relative to the beginning of the first stream*.
The first stream immediately follows the stream table, at a position known as the *origin*. The
offset of the first stream is therefore always zero, because the first data byte of the first stream
is at the origin. Other streams will have varying offsets, but they are all relative to the origin,
not to the base of the file.

For example, suppose a Confidential Package contains `5` streams. The frame header size will be `16`
bytes, because this is fixed for all Confidential Packages. The stream table has an entry for each
stream, where each entry is a pair of 64-bit (8-byte) integers. This means that the stream table
will be `5 * 2 * 8 = 80` bytes in size. This makes a total of `16 + 80 = 96` bytes for the frame
header and the stream table combined. The 97th byte of the file would therefore be the origin. This
would be where the first stream begins. Seeking to the origin would be equivalent to seeking to
position `96` relative to the base of the file. All stream offsets, in fact, can trivially be
converted into file offsets by adding `96` to them in this example. But if the file contained a
different number of streams, the origin position would likewise be different (because the stream
table size would change).

The second number in each stream table entry is simply the total size of the stream, in bytes.

## Streams

After the fixed frame header and the stream table, the streams themselves make up the entire
remaining content of the Confidential Package file. There is no additional structural information or
header data in binary format. There are no per-stream headers. All remaining data is specific to
each stream.

Each stream is addressed by a numerical index that is zero-based. The first stream therefore has an
index of `0`, the next has an index of `1` and so on. The stream table makes it possible to locate
the data for any stream, given this numeric index. It is a simple matter of consulting the
corresponding entry in the stream table.

The overall file organisation can therefore be summarised as follows, where N is the number of
streams:

`````
|   Frame Header   |  Stream Table  |  Stream 0 Data  ...  |  Stream (N-1) Data  |
|    (16 bytes)    | (N * 16 bytes) |                 ...  |                     |

                                    ^
                                    |
                                  origin
`````

A minimal valid Confidential Package file must have at least two streams: one for the manifest, and
another for the confidential payload. Typical packages would also contain additional streams to
embed other data required for processing, such as authentication tags and nonces for decryption.
Streams can also be used to embed certificates and other public resources. All additional streams
and their uses will be described in the package manifest.

Because stream indexes are represented within the file as 16-bit values, it follows that there can
be a theoretical maximum of 65,536 distinct streams within a valid Confidential Package file.
However, in practice for all expected use cases, the number of streams in the file would be probably
less than 10.

## Padding

Streams are expected to follow one another contiguously within the file. However, since the stream
table provides the offset as well as the size for each stream, this does allow for the possibility
of some padding bytes to exist between the streams, if this is convenient for alignment purposes.
Stream sizes always indicate the number of meaningful bytes of content in a stream, exclusive of any
padding. In order to fully consume a stream, it is necessary to read precisely this number of bytes
from the correct offset relative to the origin.

## Large Packages

Note that stream offsets and stream sizes are all described using 64-bit integer values. This is
designed to cater for the possibility that a Confidential Package, or particularly the confidential
payload that it contains, might be very large. This is why the file is organised in terms of
separate individual streams. It allows for both producers and consumers to process some parts of the
file independently from others, without needing to model the entire package contents in local memory
all at once.

## Package Manifest

The manifest is essentially a "map" of the Confidential Package file. It is a JSON document,
allowing it to be processed using readily-available tools and libraries. Every Confidential Package
contains exactly one manifest, and it is housed in one of the streams. The fixed frame header
indicates which stream contains the manifest. The manifest is a critical resource for processing the
rest of the file. Consumers will always locate and parse the manifest stream first.

Manifest structure and content are defined as part of this specification. The remainder of this
document will focus on different aspects of the manifest, since the manifest and the frame structure
are the only two components of the complete specification for Confidential Packages.

## Example Manifest

An example manifest document for a Confidential Package is given below. It has been pretty-printed
for clarity.

`````
{
  "cp-id": "5d286b7e-ff68-4b4b-b7b8-05f55dbfd0c7",
  "cp-name": "Fibonacci Demo Confidential Application",
  "cp-vendor": "Confidential Packaging Contributors",
  "payload": {
    "data": 0,
    "target": {
      "arch": "aarch64",
      "os": "OP-TEE"
    },
    "ver": {
      "maj": 1,
      "min": 0,
      "rev": 0,
      "date": "2021-09-12 15:48:41.691763 UTC"
    },
    "enc": {
      "aes-gcm": {
        "key_name": "5d286b7e-ff68-4b4b-b7b8-05f55dbfd0c7",
        "key_strength": 256,
        "nonce": 1,
        "tag": 2
      }
    },
    "dig": {
      "sha256": {
        "data": 3
      }
    },
    "sig": {
      "rsa-pkcs1v15": {
        "key_strength": 2048,
        "data": 4
      }
    },
    "cert": {
      "embedded-x509-pem": {
        "data": 5
      }
    }
  },
  "package": {
    "map": [
      0,
      1,
      2,
      3,
      4,
      5,
      6
    ],
    "dig": "none",
    "sig": "none",
    "cert": "none"
  }
}
`````

## Breakdown of Example Manifest

This section of the document provides an annotated breakdown of the example manifest document in
order to explain its individual elements and their meaning.

The top-level (root) document structure is as follows:

`````
{
  "cp-id": "5d286b7e-ff68-4b4b-b7b8-05f55dbfd0c7",
  "cp-name": "Fibonacci Demo Confidential Application",
  "cp-vendor": "Confidential Packaging Contributors",
  "payload": {
    ...
  },
  "package": {
    ...
  }
}
`````

The `cp-id` field provides a UUID for this Confidential Package. This string needs to be unique at
least within a given deployment where the package is being created and consumed. The specification
does not require any particular format for this string, but a string UUID is expected to be typical.

The `cp-name` and `cp-vendor` fields supply human-readable names for the package and its
supplier/vendor respectively. These fields are purely for information and are otherwise unused when
processing the package.

The `payload` structure describes the attributes of the confidential payload, and provides
information about the version, encryption method, digest and signing schemes.

The `package` structure can be used to describe a digest and signing scheme for the package as a
whole, which can be independent of the schemes used for the payload.

The structure within `payload` is as follows:

`````
  "payload": {
    "data": 0,
    "target": {
      ...
    },
    "ver": {
      ...
    },
    "enc": {
      ...
    },
    "dig": {
      ...
    },
    "sig": {
      ...
    },
    "cert": {
      ...
    }
  }
`````

The `data` field is a stream index. It defines which of the package streams contains the
confidential payload. In this example, the payload is the stream with index `0`. In other words, it
is the first stream in the package. Its offset would be `0` relative to the origin. The manifest
always references streams using their zero-based integer index. Consumers must use the stream table
to locate the stream within the package file. File offsets are never contained within the manifest.
This is a deliberate design choice, because it means that the manifest can be both consumed and
produced without knowledge of the precise file layout.

The `target` structure indicates the expected processor architecture and TEE operating system for
which the confidential payload is intended. While the Confidential Package specification itself is
deliberately architecture-neutral, the compiled payload is necessarily target-specific. The data in
this part of the manifest allows the intended target environment to be specified.

The `ver` structure provides versioning and build information. This structure follows the standard
three-component version number structure adopted by [semantic versioning](https://semver.org): major
version number, minor version number and revision/patch number. There is also a field for a build
date and timestamp to be specified.

The `enc` structure provides information about how the confidential payload has been encrypted. This
structure has variants depending on the encryption algorithm used. For AES-GCM encryption, there are
nested components with stream indexes for the nonce and authentication tag. This structure
deliberately provides no information about how or where to obtain the key for decryption. A key
identifier is provided for disambiguation purposes only. It is the responsibility of the end-to-end
deployment to determine how decryption keys are stored and distributed. This information is kept out
of the package file itself for reasons of security. In this example, the `enc` structure is as
follows:

`````
    "enc": {
      "aes-gcm": {
        "key_name": "5d286b7e-ff68-4b4b-b7b8-05f55dbfd0c7",
        "key_strength": 256,
        "nonce": 1,
        "tag": 2
      }
    }
`````

This tells us that the confidential payload has been encrypted with [AES in GCM
mode](https://en.wikipedia.org/wiki/Galois/Counter_Mode) with a 256-bit key. The name of the key is
given, but all other information about the key source is deliberately omitted. The `nonce` and `tag`
fields a stream indexes to the nonce and authentication tag vectors respectively. These vectors are
needed in order for the consumer to execute the decryption process once the key has been obtained.

The `dig` structure provides information about the digest algorithm that has been used for the
confidential payload, including a stream index for the digest itself.

The `sig` structure indicates the cryptographic scheme that was used to sign the confidential
payload, including a stream index for the resulting signature.

The `cert` structure provides information about the public certificate for the confidential payload,
which would be used to obtain the public key to verify the signature. The specification supports a
variety of embedding and referencing options for the certificate. In this example, it is specified
as follows:

`````
    "cert": {
      "embedded-x509-pem": {
        "data": 5
      }
    }
`````

This tells us that the certificate has been embedded in X509 PEM format. The inner `data` field is a
zero-based stream index, allowing the certificate to be located within the package.

Finally, the `package` structure allows for the entire package to be hashed and signed independently
of the confidential payload. Hashing and signing can occur over the content of multiple data streams
(including the manifest itself). The example manifest has the following:

`````
  "package": {
    "map": [
      0,
      1,
      2,
      3,
      4,
      5,
      6
    ],
    "dig": "none",
    "sig": "none",
    "cert": "none"
  }
`````

The `dig`, `sig` and `cert` structures are defined in the same way as for the confidential payload.
However, in this example, there is no additional hashing or signing for the package.

The `map` array would be used only when package hashing or signing are in effect, hence it is
redundant in this example. When used, it provides the sequence of stream indexes over which the
digest and signature have been computed. Consumers would concatenate the data for all of these
streams, precisely in the order given, to validate the digest and verify the signature.

## Full JSON Manifest Specification

The full manifest specification will be provided in a future version of this document.
