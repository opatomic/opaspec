# Opatomic serialization format

## Types
 - undefined
 - null
 - boolean
 - number
   - int
   - dec
   - bigint
   - bigdec
 - binary blob
 - string (utf-8 only)
 - array
 - sortmax

## varint

This is a concept that is used to help serialize many different object types. An integer is serialized
7 bits at a time starting with the least significant 7 bit chunk. The most significant bit of each byte
is set to indicate that there's another 7 bit chunk.
Refer [here](https://github.com/multiformats/unsigned-varint) for more info. This format further restricts
a varint to be 1-9 bytes long and represent an integer in the range 1 to (2^63)-1. The final byte
of a varint cannot be zero. Therefore, a varint cannot encode the number zero.

Examples:

      1 => 0x01
    127 => 0x7f
    128 => 0x80 0x01
    255 => 0xff 0x01
    300 => 0xac 0x02

## Type codes

### Constants

| byte | name         |
|------|--------------|
| 0x55 | undefined    |
| 0x4e | null         |
| 0x46 | false        |
| 0x54 | true         |
| 0x4f | zero         |
| 0x41 | empty blob   |
| 0x52 | empty string |
| 0x4d | empty array  |
| 0x5a | sortmax      |

### Numbers

| byte | name        |
|------|-------------|
| 0x44 | +int        |
| 0x45 | -int        |
| 0x47 | ++dec       |
| 0x48 | +-dec       |
| 0x49 | -+dec       |
| 0x4a | --dec       |
| 0x4b | +bigint     |
| 0x4c | -bigint     |
| 0x56 | ++bigdec    |
| 0x57 | +-bigdec    |
| 0x58 | -+bigdec    |
| 0x59 | --bigdec    |

### Variable length objects

| byte | name        |
|------|-------------|
| 0x42 | blob        |
| 0x53 | string      |
| 0x5b | array start |
| 0x5d | array stop  |

## int

An int is serialized as

    [type value]

where value is a varint. The sign is determined from the type according to the type code table above.

Example:

    The number -300 would be serialized as the following bytes:
      0x45 0xac 0x02

## dec

A dec is serialized as

    [type exponent significand]

where exponent and significand are each serialized as a varint. Together they create the formula:

    significand * (10^exponent)

The sign of the exponent and significand is determined from the type:

    | type |                        |
    |------|------------------------|
    | 0x47 | +exponent +significand |
    | 0x48 | +exponent -significand |
    | 0x49 | -exponent +significand |
    | 0x4a | -exponent -significand |

Example:

    The number 12.3 would be serialized as 123 * 10^-1 which would be the following bytes:
      0x49 0x01 0x7b

## bigint

A bigint is serialized as

    [type len bytes]

where len is the number of bytes serialized as a varint and bytes is the bigint serialized in big-endian form. The first byte
following length must not be 0. Therefore, 0 cannot be serialized as a bigint.

Example:

    The number -3735928559 would be the following bytes:
      0x4c 0x04 0xde 0xad 0xbe 0xef

## bigdec

A bigdec is serialized as

    [type exponent len bytes]

where exponent is a varint and len+bytes forms a bigint to represent the significand. Together they create the formula:

    significand * (10^exponent)

The sign of the exponent and significand is determined from the type:

    | type |                        |
    |------|------------------------|
    | 0x56 | +exponent +significand |
    | 0x57 | +exponent -significand |
    | 0x58 | -exponent +significand |
    | 0x59 | -exponent -significand |

Example:

    The number -3735928.559 would be serialized as -3735928559 * 10^-3 which would be the following bytes:
      0x59 0x03 0x04 0xde 0xad 0xbe 0xef


## blob

A zero length blob is serialized as the single byte 0x41. Any other blob is serialized as:

    [0x42 varint bytes]

where varint represents the number of bytes, and bytes represents the raw bytes.

Example:

    the byte array
      0x6f 0x70 0x61 0x74 0x6f 0x6d 0x69 0x63
    would be serialized as:
      0x42 0x08 0x6f 0x70 0x61 0x74 0x6f 0x6d 0x69 0x63

## string

A zero length string is serialized as the single byte 0x52. Any other string is serialized as:

    [0x53 varint chars]

where varint represents the number of chars, and chars is the sequence of UTF-8 characters.

Example:

    The string "opatomic" would be serialized as the following bytes:
      0x53 0x08 0x6f 0x70 0x61 0x74 0x6f 0x6d 0x69 0x63

A string must be valid UTF-8.

## array

An empty array is serialized as the single byte 0x4d. Any other array starts with the byte 0x5b
and ends with 0x5d. It can contain any number of objects including child arrays.


