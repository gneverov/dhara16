# Dhara16
Dhara16 is a modified version of [Dhara](https://github.com/dlbeer/dhara) for small NOR flash chips commonly used as XIP memory for microcontrollers. The primary modification is to use a 16-bit radix instead of 32 bits. This reduces the overhead of metadata to facilitate filesystems of the order of a few megabytes.

Given a NOR flash with 256-byte program blocks, 4096-byte erase blocks, and 1 MB total size. It is most efficient to use a Dhara page size of 512 bytes, giving 8 pages per erase block, and an efficiency of 7/8. Combined with a GC ratio of 6, the overall efficiency is 3/4. So of the 1024 kB of flash memory, only 768 kB is usable by a filesystem.

Whether this cost is worthwhile depends on the application. If you have a lot of files or do a lot of writes, then using a flash translation layer might be a benefit. In addition to wear-levelling, using Dhara also reduces the filesystem block size from 4096 bytes to 512 bytes, thus reducing internal fragmentation which offsets the cost of the flash translation layer.

# Dhara
```
	   Dhara: NAND flash translation layer for small MCUs

		     Daniel Beer <dlbeer@gmail.com>
			       1 Apr 2017

Dhara is a small flash translation layer designed to be used in
resource-constrained systems for managing NAND flash. It provides a
mutable block interface with standard read and write operations. It has
the following additional features:

  * Perfect wear-levelling: the erase count of any two blocks will
    differ by at most 1.

  * Trim: logical sectors can be deleted to improve performance if their
    content is not required.

  * Data integrity: write() (and trim()) of logical sectors are
    atomic. If the power fails, the state rolls back to the last
    synchronization point. Synchronization points occur at regular
    intervals, but can also be reached on demand.

  * Real-time performance: all operations, including startup, are O(log
    n) worst case in the size of the chip, if bad blocks are uniformly
    distributed.

The implementation makes minimal assumptions regarding the underlying
NAND chip. It can be used with almost any chip available, and can take
advantage of extra hardware features where available. In particular:

  * No OOB data is consumed. All available OOB bytes can be spent on
    ECC.

  * It can take advantage of internally buffered copy operations, if the
    NAND chip supports these.

  * It can make use of NAND chip's on-board ECC, if available. If
    software ECC is required, implementations of Hamming and BCH codes
    are provided in the source distribution (see the ecc/ directory).

  * It does not require partial page programmability. However, if the
    functionality is present, then it can be taken advantage of by
    presenting a smaller pseudo-page size by dividing the real page size
    by the number of allowed reprogram operations. The only restriction
    is that partial writes must contain complete ECC information.

  * It will take advantage of the ability to do partial reads if this is
    possible. Reads must be error checked and corrected (this can
    usually be done in units smaller than the page size).

The implementation consists of the files in the dhara/ subdirectory,
plus the NAND layer implementation, which you must provide. The
top-level interface is the set of functions described in map.h (see the
comments in the header file for more details):

    init: initialize a map layer instance
    resume: scan the map and recover the saved state
    clear: delete all data
    capacity, size: obtain usage statistics
    find: obtain the physical location of a logical sector
    read: read a logical sector
    write: write a logical sector
    copy_page: copy a raw flash page to a logical sector
    copy_sector: copy one logical sector to another
    trim: remove a logical sector from the map
    sync: ensure that changes to the map are committed
    gc: manually trigger garbage collection

To provide the NAND layer, implement the set of functions described in
nand.h (see comments for details). In summary, you must provide the
following operations:

    is_bad: determine whether a block is bad
    mark_bad: mark a NAND block as bad
    erase: erase a NAND block
    prog: program a NAND page, including ECC and checksums
    is_free: determine whether a page is erased (unprogrammed)
    read: read a (possibly partial) NAND page, and attempt ECC if
      necessary
    copy: copy one page to another, using internal buffers if possible

Check the datasheet for your chip for information on these operations.
In most cases, the manufacturer will specify a preferred layout scheme
for the ECC and bad block markers in the OOB region. Pay particular
attention to the marker used for factory-marked bad blocks!

If your ECC scheme is such that programming an all-0xff page is
equivalent to a no-op, then it's ok for your implementation of is_free()
to simply check for all-0xff page content.

Note that bad blocks need only be queried one at a time. It's not
necessary to maintain a bad-block table -- just the standard OOB marking
scheme is fine, and preserves the performance guarantees of the map
layer.

Also note that when implementing partial read, you must read enough of
the page that you're able to apply ECC and check for uncorrectable
errors. Uncorrectable errors *must* be detected in order for the data
integrity guarantees to be valid. This may require the use of a checksum
in addition to ECC. If this is done, ensure that the checksum bytes are
also protected by ECC!

Implementations of two popular ECC mechanisms (Hamming and BCH) can be
found in the ecc/ subdirectory. Each implements ECC over variable-sized
chunks (256 or 512 bytes are typical sizes). Multiple ECC chunks may be
required per page.
```