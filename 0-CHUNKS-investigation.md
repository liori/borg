# BorgBackup Chunks Investigation Report

## Executive Summary

This report documents how to iterate over all chunks in a BorgBackup repository and extract the required information for creating a chunk dump command. The goal is to produce a CSV file with: chunk ID, archive name, compressed size, uncompressed size, file path, and offset within the file.

## Key Findings

### 1. Chunk Data Structure

BorgBackup represents chunks using the `ChunkListEntry` namedtuple defined in `src/borg/item.pyx:133`:

```python
ChunkListEntry = namedtuple('ChunkListEntry', 'id size csize')
```

**Fields:**
- `id` (bytes): 32-byte chunk identifier (SHA256 hash or similar)
- `size` (int): Uncompressed size in bytes
- `csize` (int): Compressed size in bytes

### 2. Architecture: Two-Layer Chunking

BorgBackup uses a hierarchical chunking architecture:

#### Layer 1: Meta-Chunks (Archive Metadata)
- Archives store their file listings as serialized Item objects
- These Item objects are themselves chunked and stored in the repository
- Reference: `archive.metadata.items` contains list of chunk IDs for the metadata
- The `archive.iter_items()` method transparently unpacks these meta-chunks

#### Layer 2: Data-Chunks (File Content)
- Individual files are split into chunks during backup
- Each file's Item object contains a `chunks` list with all its data chunks
- Reference: `item.chunks` is a list of `ChunkListEntry` objects
- Chunks are stored sequentially in the order they appear in the file

### 3. Core Components and File Locations

#### Item Class (`src/borg/item.pyx`)
- **Line 133**: `ChunkListEntry` definition
- **Line 135-199**: `Item` class - represents a file/directory in an archive
- **Line 186**: `chunks` property - list of ChunkListEntry objects for file data
- **Line 182**: `size` property - total uncompressed size (sum of chunk sizes)

#### Archive Class (`src/borg/archive.py`)
- **Line 425**: `Archive` class definition
- **Line 436-464**: `__init__` method - initialization with repository, key, manifest
- **Line 592-599**: `iter_items()` method - **KEY METHOD** for iterating over all items in archive
  ```python
  def iter_items(self, filter=None, partial_extract=False, preload=False, hardlink_masters=None):
      for item in self.pipeline.unpack_many(self.metadata.items, ...):
          yield item
  ```

#### Manifest and Archives (`src/borg/helpers/manifest.py`)
- **Line 38-105**: `Archives` class - manages the collection of archives
- **Line 49-50**: `__iter__` method - allows iteration over archive names
- **Line 78-105**: `list()` method - returns list of ArchiveInfo objects with filtering/sorting

### 4. Iteration Strategy

To iterate over all chunks, use this nested loop structure:

```python
# Pseudo-code for chunk iteration
for archive_info in manifest.archives.list(sort_by=['ts']):
    archive_name = archive_info.name
    archive = Archive(repository, key, manifest, archive_name, cache=cache)

    for item in archive.iter_items():
        # Only files have chunks; directories, symlinks, etc. don't
        if 'chunks' in item:
            file_path = item.path

            for chunk_index, chunk_entry in enumerate(item.chunks):
                chunk_id = chunk_entry.id           # bytes (32 bytes)
                compressed_size = chunk_entry.csize  # int
                uncompressed_size = chunk_entry.size # int

                # Calculate offset: sum of all previous chunk sizes
                offset = sum(c.size for c in item.chunks[:chunk_index])

                # Now we have all required data for CSV row:
                # - chunk_id (convert to hex for readability)
                # - archive_name
                # - compressed_size
                # - uncompressed_size
                # - file_path
                # - offset
```

### 5. Practical Example from Codebase

The `do_list` command in `src/borg/archiver.py:1416` shows real-world usage:

```python
def _list_archive(self, args, repository, manifest, key):
    matcher = self.build_matcher(args.patterns, args.paths)

    def _list_inner(cache):
        archive = Archive(repository, key, manifest, args.location.archive, cache=cache,
                          consider_part_files=args.consider_part_files)

        formatter = ItemFormatter(archive, format, json_lines=args.json_lines)
        for item in archive.iter_items(lambda item: matcher.match(item.path)):
            sys.stdout.write(formatter.format_item(item))
```

### 6. Important Implementation Notes

#### Repository and Manifest Setup
Commands typically use decorators like `@with_repository` that provide `repository`, `manifest`, and `key` objects. For a new command, you'll need:

1. Open repository connection
2. Load manifest (contains archive list)
3. Access encryption key
4. Optionally use cache for performance

#### Item Types
Not all items have chunks:
- **Regular files**: Have `chunks` list
- **Directories**: No chunks (only metadata)
- **Symlinks**: No chunks (target stored in metadata)
- **Special files** (devices, fifos): No chunks

Always check `if 'chunks' in item` or `if item.chunks` before accessing the chunks list.

#### Chunk Deduplication
The same chunk ID can appear in:
- Multiple files within the same archive
- Multiple files across different archives
- Multiple positions within the same file (if file has repeated content)

For a complete chunk dump, you'll want to output **every occurrence** of each chunk (not deduplicate), since each occurrence has different context (archive name, file path, offset).

#### Chunk Index in Repository
The repository maintains a `ChunkIndex` (`src/borg/hashindex.pyx:277`) that maps chunk IDs to `(refcount, size, csize)`. However:
- The `ChunkListEntry` in `item.chunks` already contains all needed size information
- No need to query the repository's ChunkIndex for basic chunk metadata
- The repository index is primarily used for deduplication and garbage collection

### 7. Command Structure Recommendation

Based on the codebase patterns, a new `borg chunks-dump` command should:

1. **Accept repository location**: `borg chunks-dump REPOSITORY`
2. **Optional output file**: `--output FILE` (default: chunks-dump.csv.xz)
3. **Optional archive filtering**: `--glob-archives PATTERN` to limit which archives to scan
4. **Optional progress indication**: `--progress` to show scanning status
5. **Use decorators**: `@with_repository` and `@with_cache` from `archiver.py`

#### High-Level Implementation Structure

```python
def do_chunks_dump(self, args, repository, manifest, key):
    """Dump all chunks information to CSV file"""

    import csv
    import lzma
    from binascii import hexlify

    # Open output file with xz compression
    with lzma.open(args.output, 'wt', newline='') as csvfile:
        writer = csv.writer(csvfile)
        writer.writerow(['chunk_id', 'archive_name', 'compressed_size',
                        'uncompressed_size', 'file_path', 'offset'])

        def process_archives(cache):
            # Get list of archives to process
            archive_list = manifest.archives.list(
                glob=args.glob_archives,
                consider_checkpoints=args.consider_checkpoints,
                sort_by=['ts']
            )

            for archive_info in archive_list:
                archive_name = archive_info.name
                archive = Archive(repository, key, manifest, archive_name, cache=cache)

                for item in archive.iter_items():
                    if item.chunks:
                        file_path = item.path

                        for chunk_index, chunk_entry in enumerate(item.chunks):
                            chunk_id_hex = hexlify(chunk_entry.id).decode('ascii')
                            compressed_size = chunk_entry.csize
                            uncompressed_size = chunk_entry.size
                            offset = sum(c.size for c in item.chunks[:chunk_index])

                            writer.writerow([
                                chunk_id_hex,
                                archive_name,
                                compressed_size,
                                uncompressed_size,
                                file_path,
                                offset
                            ])

        # Use cache if available (for performance)
        cache_if_remote(repository, decrypted_cache=process_archives)
```

### 8. Performance Considerations

#### Memory Usage
- `archive.iter_items()` is a generator - memory efficient
- Each item is processed one at a time
- Chunk list for each item is loaded into memory (proportional to file size / chunk size)
- For very large files (millions of chunks), this should still be manageable

#### I/O Performance
- Reading archive metadata requires repository access
- Each archive's metadata chunks must be fetched and decrypted
- Use cache when available (`cache_if_remote` pattern)
- Consider `--progress` to show activity during long operations

#### Estimated Output Size
- Typical chunk: ~2 MB uncompressed (default)
- CSV row: ~150 bytes (chunk_id_hex=64, paths can be long)
- XZ compression: expect 10-20x compression ratio
- For 1 TB of backed-up data: ~500,000 chunks → ~75 MB CSV → ~4-8 MB compressed

### 9. Testing Strategy

When implementing the command, test with:

1. **Empty repository**: Should produce header-only CSV
2. **Single archive, single small file**: Verify chunk details are correct
3. **Multiple archives with deduplicated content**: Verify chunks appear multiple times
4. **Large files with many chunks**: Verify offset calculations
5. **Mixed content** (files, directories, symlinks): Verify only files with chunks are included
6. **Archive filtering**: Test `--glob-archives` pattern matching

### 10. Open Questions and Considerations

#### Checkpoint Archives
- Archives with `.checkpoint` in name are incomplete backups
- Should these be included by default? Probably add `--consider-checkpoints` flag
- See how other commands handle this (e.g., `manifest.archives.list(consider_checkpoints=False)`)

#### Sparse Files
- Do sparse file chunks have special handling?
- Need to verify if offset calculation is affected

#### Hardlinked Files
- Multiple items might reference same content
- Current approach will output chunks for each hardlink occurrence
- This is probably desired behavior (shows all references)

#### Chunk Ordering
- Within a file: chunks are in sequential order (verified by offset calculation)
- Across files: depends on backup order
- Across archives: chronological by archive timestamp (if using `sort_by=['ts']`)

## Conclusion

The required information for chunk dumping is readily accessible through BorgBackup's existing API:

1. **Iterate archives**: `manifest.archives.list()`
2. **Iterate items**: `archive.iter_items()`
3. **Access chunks**: `item.chunks` (list of ChunkListEntry)
4. **Extract data**: All fields available directly (id, size, csize) plus item.path

The implementation is straightforward and follows established patterns in the codebase. The main work is properly handling the decorators, cache management, and output file formatting.

## Next Steps

1. Implement the `do_chunks_dump` command in `src/borg/archiver.py`
2. Add argparse configuration for command-line arguments
3. Add appropriate decorators (`@with_repository`, etc.)
4. Implement CSV writing with XZ compression
5. Add progress indication for long-running operations
6. Write tests in `src/borg/testsuite/`
7. Update documentation and command help text
