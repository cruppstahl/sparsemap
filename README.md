
`sparsemap` is a sparse, compressed bitmap. In best case, it can store 2048
bits in just 8 bytes. In worst case, it stores the 2048 bits uncompressed and
requires additional 8 bytes of overhead.

The "best" case happens when large consecutive sequences of the bits are
either set ("1") or not set ("0"). If your numbers are consecutive 64bit
integers then sparsemap can compress up to 16kb in just 8 bytes.

## How does it work?

On the lowest level, bits are stored in BitVectors (a uint32_t or uint64_t).

Each BitVector has an additional descriptor (2 bits). All descriptors are
stored in a single Word which is prepended to the BitVectors. (The descriptor
Word and the BitVectors have the same size.) The descriptor of a BitVector
specifies whether the BitVector consists only of set bits ("1"), unset
bits ("0") or has a mixed payload. In the first and second case the
BitVector is not stored.

An example shows a sequence of 4 x 16 bits (here, each BitVector and the
Descriptor word has 16 bits):

      Descriptor:
      00 00 00 00 11 00 11 10
      ^^ ^^ ^^ ^^-- BitVector #0 - #3 are "0000000000000000"
                  ^^-- BitVector #4 is "1111111111111111"
                     ^^-- BitVector #5 is "0000000000000000"
                        ^^-- BitVector #7 is "1111111111111111"
                           ^^-- BitVector #7 is "0110010101111001"

Since the first 7 BitVectors are either all "1" or "0" they are not stored.
The actual memory sequence looks like this:

      0000000011001110 0110010101111001

Instead of storing 8 Words (16 bytes), we only store 2 Words (2 bytes): one
for the Descriptor, one for last BitVector #7.

Since such a construct (it's called a MiniMap) has a limited capacity, another
structure is created on top of it, a `Sparsemap`. The Sparsemap stores a list
of MiniMaps, and for each MiniMap it stores the absolute address. I.e. if
the user sets bit 0 and bit 10000, and the MiniMap capacity is 2048, the
Sparsemap creates two MiniMaps. The first starts at offset 0, the second starts
at offset 8192.

# Usage instructions

The file `main.cc` has example code. Here is a small excerpt:

    // we need a buffer to store the SparseMap
    unsigned char buffer[1024];

    sparsemap::SparseMap<uint32_t, uint64_t> sm;
    sm.create(buffer, sizeof(buffer));

    // after initialization, the used size is just 4 bytes (sizeof(uint32_t))
    assert(sm.get_size() == sizeof(uint32_t));

    // set the first bit
    sm.set(0, true);

    // check that the first bit was set
    assert(sm.is_set(0) == true);

    // unset the first bit
    assert(sm.is_set(1) == false);

    // check that the first bit is now no longer set
    sm.set(0, false);

## Final words

This bitmap implementation has very efficient compression when using on long
sequences of set (or unset) bits. I.e. with a word size of 64bit, and a
payload of consecutive numbers without gaps, the payload of 2048 x
sizeof(uint64_t) = 16kb can be stored in just 8 bytes!

However, if the sequence is not consecutive and has gaps, it's possible that
the compression is completely inefficient, and the size basically is identical
to an uncompressed bitvector (even higher because a few bytes are required for
metadata). In such cases, other compression schemes are more efficient (i.e.
http://lemire.me/blog/archives/2008/08/20/the-mythical-bitmap-index/).

This library was originally created for hamsterdb [http://hamsterdb.com] in
order to compress the key indices. For several technical reasons this turned
out to be impossible, though. (If you're curious then feel free to drop
me a mail.) I'm releasing it as open source, hoping that others can make good
use of it.

