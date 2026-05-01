# btrfs-snapshot-dedup

There are cases where btrfs snapshots lose their shared extents with the original file. This causes the data to be stored twice — once in the live file, once in each snapshot — wasting significant disk space.

This tool re-consolidates snapshot extents with the live file. Fast, metadata-only — no hashing, no data reads. Just stat + FIEMAP + FIDEDUPERANGE.

### Known causes of broken extent sharing

- **Defragmentation** — `btrfs filesystem defragment` rewrites extents, snapshots keep the old ones
- **Post-compression** — `btrfs filesystem defragment -czstd` creates new compressed extents, snapshots still reference the old uncompressed data
- **bees dedup daemon** — deduplicates live files but does not consolidate snapshot copies ([Zygo/bees#342](https://github.com/Zygo/bees/pull/342))

## How It Works

For each file in the live subvolume:

1. **Binary search** across snapshots to find the oldest copy (stat only)
2. **Size check** — if size differs, skip
3. **FIEMAP check** — compare physical extent offsets between live and snapshot
   - Same offset → already shared → skip
   - Different offset → candidate
4. **Collect group** — all snapshot copies + logical-resolve for other references
5. **FIDEDUPERANGE** — deduplicate in one multi-dest ioctl call (up to 120 dests per call)

The kernel verifies each deduplication byte by byte — metadata comparison is only used to find candidates.

## Performance

**Slow machine (ARM SBC, 4 cores, 22 TB RAID0, ~1000 snapshots):**

| Subvolume | Files | Candidates | Copies | Saved | Runtime |
|-----------|-------|-----------|--------|-------|---------|
| media (large files) | 11,684 | 23 | 1,590 | 7.68 GiB | 12m05s |
| package-cache | 82,498 | 1,493 | 157,267 | 4.93 GiB | 1h05m |

**Fast machine (x86, 4 cores, 22 TB RAID1, ~500 snapshots):**

| Subvolume | Files | Candidates | Copies | Saved | Runtime |
|-----------|-------|-----------|--------|-------|---------|
| media (large files) | 11,697 | 6 | 83 | 0 B | 1m38s |
| package-cache | 83,380 | 69 | 2,096 | 15.5 MiB | 22m30s |
| file-sync | 766,009 | 43 | 1,634 | — | ~15min |

Comparison with bees:

| | bees | btrfs-snapshot-dedup |
|---|---|---|
| Detection | Block-hash comparison (reads data) | stat() metadata (no reads) |
| Scan | Sequential extent-tree (22 TB) | filepath.WalkDir per subvol |
| Duration | Months for full scan | Minutes per subvolume |
| Dedup | 1:1 per pair | Multi-dest (up to 120 at once) |

## Requirements

- Linux with btrfs filesystem
- Go 1.22+ with cgo (uses kernel headers for ioctl definitions)
- Root access (FIEMAP and FIDEDUPERANGE require it)
- `btrfs` command-line tools (for `inspect-internal subvolid-resolve` and `logical-resolve`)

## Build

```bash
go build -o btrfs-snapshot-dedup btrfs-snapshot-dedup.go
```

## Usage

```bash
sudo ./btrfs-snapshot-dedup [options] <mountpoint> <subvolume> [find-filter...]
```

Everything after `<mountpoint> <subvolume>` is passed directly to `find(1)` as filter arguments. Default (no filter): `find <path> -type f` (all files).

Examples:

```bash
# Dedup all files
sudo ./btrfs-snapshot-dedup /mnt/btrfs mysubvol

# Only large files (>= 100MB)
sudo ./btrfs-snapshot-dedup /mnt/btrfs mysubvol -size +100M

# Only text/config files
sudo ./btrfs-snapshot-dedup /mnt/btrfs mysubvol '(' -iname '*.txt' -o -iname '*.json' -o -iname '*.xml' ')'

# Combine size + extension filter
sudo ./btrfs-snapshot-dedup /mnt/btrfs mysubvol -size +1M '(' -iname '*.log' -o -iname '*.txt' ')'

# Resume after interrupt
sudo ./btrfs-snapshot-dedup -start-at 'path/to/last/file' /mnt/btrfs mysubvol
```

### Options

| Option | Default | Description |
|--------|---------|-------------|
| `-workers` | `4` | Number of parallel FIDEDUPERANGE workers |
| `-start-at` | (none) | Resume: skip files before this path (lexicographic) |

## Output

### Live status (stderr, every 10 seconds)

```
[3m40s] found=9827 buf=1857 checked=9827 shared=9799 not_found=0 changed=26 candidates=1/66 pending=0 deduped=1 saved=4.2M
```

Fields follow a funnel order:

| Field | Meaning |
|-------|---------|
| `found` | Files matching size filter (walker progress) |
| `buf` | Files in queue waiting to be processed |
| `checked` | Files processed (with ETA) |
| `shared` | Already shares extents with snapshot (skipped) |
| `not_found` | File not found in any snapshot (skipped) |
| `changed` | File size differs from snapshot (skipped) |
| `candidates` | Duplicate groups found / total copies across all snapshots |
| `pending` | Groups waiting for a dedup worker |
| `deduped` | Groups successfully deduplicated |
| `saved` | Bytes saved (unique extents freed × file size) |

### Crash-safe output files

- **`candidates.fdupes`** — all found candidate groups, written immediately (fdupes format)
- **`candidates.done`** — successfully deduplicated groups (same format)

On interrupt (Ctrl+C), the last processed file path is shown for use with `-start-at`.

## Background: Why Snapshots Don't Follow

### Use Case 1: Dedup daemon catchup

When bees deduplicates file A and file B to share the same extent, it does not consolidate their snapshot copies. With 1000 snapshots, this means 1000 separate extents that are byte-identical but not shared. bees will eventually find them through its sequential extent-tree scan, but on a 22 TB filesystem this takes months.

A patch to fix this in bees has been submitted: [Zygo/bees#342](https://github.com/Zygo/bees/pull/342). This tool catches up on the historical backlog.

### Use Case 2: Post-compression reclaim

After running `btrfs filesystem defragment -czstd /path`, each file gets new compressed extents (e.g. 3 GiB → 1.1 GiB). But every snapshot still references the old 3 GiB uncompressed extents. Net result: 1.1 + 3.0 = 4.1 GiB — worse than before compression.

Running `btrfs-snapshot-dedup` after compression points all snapshots to the compressed extents, freeing the old uncompressed ones. Use find filter syntax to target only compressible file types:

```bash
sudo ./btrfs-snapshot-dedup /mnt/btrfs mysubvol '(' -iname '*.txt' -o -iname '*.json' -o -iname '*.xml' -o -iname '*.csv' -o -iname '*.log' -o -iname '*.md' -o -iname '*.yaml' -o -iname '*.sh' -o -iname '*.py' ')'
```

## Technical Notes

### FIDEDUPERANGE on read-only snapshots

`FIDEDUPERANGE` works on read-only snapshots when dest files are opened with `O_RDONLY`. Opening with `O_RDWR` fails with `EROFS`. This is the same reason `duperemove --fdupes` works on snapshots but `duperemove -dr` doesn't (in versions before v0.15).

### Accurate saved calculation

The kernel reports `bytes_deduped > 0` even when files already share the same extent (no-op). Additionally, multiple snapshot copies often share the same physical extent (COW), so deduplicating N snapshots from extent B to extent A only frees one copy of B, not N copies. This tool counts unique physical offsets among dests to report accurate savings.

## Related

- [bees](https://github.com/Zygo/bees) — btrfs deduplication daemon
- [Zygo/bees#342](https://github.com/Zygo/bees/pull/342) — Patch to add snapshot consolidation to bees
- [duperemove](https://github.com/markfasheh/duperemove) — Hash-based btrfs deduplication tool

## License

GPL-2.0 (same as bees)
