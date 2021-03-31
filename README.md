# LiTS - Lightweight/Libre/Little Time Series Format

This repository is used to sketch some ideas for a time-series data storage format that can be implemented on microcontrollers with a few kB of RAM and less than 1 MB of flash memory.

## Assumptions

- It is very probable that the time interval between data points is similar or even constant
- Data points don't change with steep steps (delta-of-delta encoding suitable)
- The timestamp is steadily increasing
- Time series of integers can be compressed much more efficiently than float, so float variables should be stored as scaled integers if possible

## Implementation ideas

- Use [Run-length encoding (RLE)](https://en.wikipedia.org/wiki/Run-length_encoding) to consume almost no space for unchanged values
- The format should be 8-bit aligned for easier computation
- 0x00 is a break sign (used to start a new block of data)
- The data type of one time series is stored in the header or file name and does not change

Below sections describe the encoding of a new measurement for the four different identified "scenarios". The "scenario" or type is stored in the first two bits, followed by actual data depending on the type.

If a varint as mentioned below does not fit into the available bits, another byte is appended to continue the varint data.

### 1. Time interval constant and value unchanged (least space-consuming)

    00nnnnnn
    n: unsigned varint specifying number of subsequent same samples

RLE is used to store the number of same samples. This allows to store slowly changing values or simple boolean on/off states very efficiently. Even strings like the git commit hash of the used software can be stored as a time series, only consuming a few bytes for every "event" (change of firmware in this case).

### 2. Time interval constant, but value changed (second-least space-consuming)

Integer values are stored with delta-of-delta encoding (see Gorilla paper [1]).

    01xxxxxx xxxxxxxx
    x: zig-zag varint containing delta-of-delta for value

Floating point variables are XOR'ed with previous value (similar to Gorilla paper) and afterwards only the non-zero bytes are stored according to below method.

    01yyyzzzz xxxxxxxx
    y: number of leading 0-bytes after XOR
    z: number of trailing 0-bytes after XOR
    x: XOR'ed payload data without 0-bytes

### 3. Time interval changed, but value the same

In this case we don't need to store dhe value, but only the changed delta-of-delta.

    10tttttt
    t: zig-zag varint containing delta-of-delte for timestamp

### 4. Both changed

This scenario allows only little compression compared to above ones.

Integers are stored similar to above, but with a full byte available for the first part of the varint.

    11tttttt xxxxxxxx
    t: zig-zag varint containing delta-of-delta for timestamp

Also floats are stored similar to the method described above.

    11tttttt 01yyyzzz xxxxxxxx
    see above

## Storage of time series metadata

Two options are possible: Information about data type, scaling and time stamp resolution are stored in the file name or in a header (first few bytes of the series).

Most suitable option still t.b.d.

## File name conventions

`device-id/Bat_V.f10s.lits`

- Tags are part of the file path
- Time interval and data type is part of the file name
  - First letter: d | f | i | b (for double, float, int, boolean types)
  - Timestamp interval: d | h | m | s | ms | us | ns (e.g. 10s, 100ms, 1h, 20m)

Default for JSON input: `Bat_V.i1s.lits`

- value precision derived from digits behind the comma
- number format: int
- Timestamp interval: 1s

### Header

Magic bytes

    4c 69 54 53 = LiTS

Time resolution

    vvveeuuu

    v: file format version: 001
    u: unit: 0000 = ps, 0001 = ns, 0010 = us, 0011 = ms, 0100 = s, 0101 = min, 0110 = h, 0111 = d, 1000 = wk, 1001 = m, 1010 = y

    e: exponent (base 10): 00 = 0, 01 = 1, 10 = 2, 11 = non-second units
    u: unit: 000 = fs, 001 = ps, 010 = ns, 011 = us, 100 = ms, 101 = s
            000 = min, 001 = h, 010 = d, 011 = wk, 100 = m, 101 = y

Value format and resolution

    1 byte

    -126 .. 126: integer with encoded exponent of base 10
    -127: float
    +127: string

# References

[1] Gorilla: A Fast, Scalable, In-Memory Time Series Database, http://www.vldb.org/pvldb/vol8/p1816-teller.pdf
