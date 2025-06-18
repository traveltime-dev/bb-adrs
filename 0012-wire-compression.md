# Buildbarn Architecture Decision Record #12: Over-the-wire Compression

Author: Tyler French<br/>
Date: 2025-06-17

# Context

The REv2 ([Remote Execution API v2](https://github.com/bazelbuild/remote-apis/blob/main/build/bazel/remote/execution/v2/remote_execution.proto#L2013)) specification supports Compressor.ZSTD for ByteStream operations. This allows clients like Bazel to upload and download artifacts in a compressed format, which can significantly reduce network bandwidth usage and improving transfer times for compressible blobs (e.g., large binaries, JARs, source tarballs). This can be hugely impactful for **users with poor connection or expensive data transfer costs**.

Remote Cache Compression has not yet been supported in Buildbarn, most notably because of 2 reasons:

1. Streaming Decompression for Random Access: The blobstore abstraction in Buildbarn uses a `BlobAccess` interface that returns `Buffer` objects. The `Buffer` interface supports [`io.ReaderAt`](https://pkg.go.dev/io#ReaderAt), which implies efficient random access. A compressed [ZSTD decoder](https://pkg.go.dev/github.com/klauspost/compress/zstd#Decoder) stream is inherently sequential. You cannot seek to an arbitrary offset in the uncompressed data without decompressing all preceding data.
2. Unknown Stream Size: When a client uploads a compressed stream, the server only knows the uncompressed size from the digest `<hash>/<size>`. The total size of the incoming compressed stream is unknown until the client finishes, making progress tracking and resource allocation difficult.

## Goals

- Implement server-side support for ZSTD compression and decompression for the ByteStream gRPC service.
- Must be memory-efficient, avoiding buffering entire large objects in memory on the server.
- The solution should integrate cleanly with Buildbarn's existing blobstore and Buffer abstractions.

## Out of Scope: At-Rest Compression

This ADR focuses exclusively on over-the-wire compression as defined by the REv2 API. Compression of data at rest within Buildbarn's storage backends is a separate, more complex problem. It would require significant changes to block-level storage logic and ensuring that all components (including workers) can handle both compressed and uncompressed data formats transparently. Since we don't know the full size of what the compressed file will be, deciding where to place files for at-rest compression is much more difficult.

# Implementation Architecture

The solution implements compression at the ByteStream server level using a streaming approach where compressed data flows directly through zstd decoder/encoder wrappers without intermediate accumulation.

## ByteStream vs Buffer Level Implementation

Compression is implemented at the ByteStream service layer rather than Buffer abstraction layer. Since the Buffer interface supports `io.ReaderAt` for random access but compressed streams are sequential, implementing at the ByteStream level handles the sequential nature naturally while keeping existing Buffer abstractions unchanged.

## Streaming Architecture

```
Write Path (Upload):
gRPC Stream → writeStreamReader → [Compressed?] → BlobStore
                                      ↓
                                 ZSTD: ZstdReader(writeStreamReader)
                                 IDENTITY: writeStreamReader
                                      ↓
                            buffer.NewCASBufferFromReader

Read Path (Download):
BlobStore → ChunkReader → [Compression Layer] → gRPC Stream
                             ↓
                        ZSTD: ZstdWriter wraps ChunkReader
                        IDENTITY: ChunkReader streams directly
                             ↓
                      readStreamWriter
```

This streaming approach requires no intermediate buffering - compressed data flows directly through decoder/encoder wrappers without accumulating in memory. Memory usage is bounded by the zstd library's internal buffers (~1-8MB) rather than total stream size, keeping compression only during transit with no additional storage layer considerations.

# Configuration

Users configure compression support through the `supportedCompressors` field in their bb_storage configuration. This impacts the `CacheCapabilities` response from the remote cache.

```jsonnet
{
  // ... other configuration ...
  supportedCompressors: [
    'IDENTITY',  // Always implied per REv2 specification
    'ZSTD',      // Enable ZSTD compression
  ],
  // ... other configuration ...
}
```

Available compressors: `IDENTITY` (uncompressed, always supported), `ZSTD` (recommended).
