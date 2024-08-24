## To run the test

```
protoc --go_out=. --go_opt=paths=source_relative encoding.proto
```

## Resource for Encoding
https://protobuf.dev/programming-guides/encoding

## What is the "Wire Format"?
The wire format is the binary encoding format that Protobuf uses to serialize the data defined in your .proto file before it's 
sent across the network (the "wire") or stored on disk.

## Protoscope to Describe the low-level Wire Format

Protoscope is a very simple language for describing snippets of the low-level wire format, which we’ll use to provide a visual reference for the encoding of various messages. Protoscope’s syntax consists of a sequence of tokens that each encode down to a specific byte sequence.

## A Simple Example

Let's use a simple example to illustrate how much space a serialized Protobuf message might
consume on disk.

```
message Test {
  optional int32 a = 1;
}
```

if you set `a = 150`, the wire format would look like this:

```
08 96 01
```

Here's the breakdown:

```
08 (1 byte): This encodes the field number (`1`) and the wire type (`variant`)
96 01 (2 bytes): This is the variable-length encoding of the integer `150`
```

So, this message consumes 3 bytes in total.

## How do we figure it out its 150?

First you drop the MSB from each byte, as this is just there to tell us whether we’ve reached the end of the number (as you can see, it’s set in the first byte as there is more than one byte in the varint). These 7-bit payloads are in little-endian order. Convert to big-endian order, concatenate, and interpret as an unsigned 64-bit integer:

```
10010110 00000001        // Original inputs.
 0010110  0000001        // Drop continuation bits.
 0000001  0010110        // Convert to big-endian.
   00000010010110        // Concatenate.
 128 + 16 + 4 + 2 = 150  // Interpret as an unsigned 64-bit integer.
```

## Base 128 Variants

The term "Base 128" in Base 128 Varints refers to the fact that each byte in the encoding can represent 128 different values (2<sup>7</sup> = 128) for the actual data, with the most significant bit (MSB) used as a continuation flag. 
This allows for efficient encoding of integers using a variable number of bytes.

0000000 (which is 0 in decimal)
to 1111111 (which is 127 in decimal).

In Base 128 Varint encoding, we only use 7 of these 8 bits to represent the actual data.
The remaining 1 bit (the most significant bit, or MSB) is used as a continuation flag.

### Continue to Base 128 Variants

The varint encoding of 150 is 0x96 0x01, which is 9601 in hexadecimal.

When it's marshalled to binary, it uses base 128 varints to encode the go struct to binary.

Variable-width integers, or varints, are at the core of the wire format. 

Variable-width integers (also known as variable-length integers or varints) are a way to encode integers using a variable number of bytes, rather than a fixed number. 
This encoding is commonly used in data formats where space efficiency is critical.

### More about Variable-with integers

Variable-length encoding format, often used in protocols like Protocol Buffers (Protobuf) to optimize the storage of data. Specifically, it's describing a method where small unsigned 64-bit integers 
can be encoded in fewer bytes, while larger numbers may take more bytes.

Here's how it works:
Variable-Length Encoding: The idea is to save space when encoding numbers by using fewer bytes for smaller values. Instead of always using 8 bytes to store a 64-bit integer, variable-length encoding adapts to the size of the number.

Small Values, Fewer Bytes: If the number is small, it can be represented using fewer bytes. For example, a number like 5 can be encoded in just 1 byte. On the other hand, a larger number like 1,000,000 might require more bytes, possibly 3 or 4.

Up to Ten Bytes for the Largest Values: For very large numbers, such as the maximum value of an unsigned 64-bit integer (18,446,744,073,709,551,615), the encoding may use up to 10 bytes.

This is efficient because most real-world numbers are small, so using fewer bytes reduces storage size and speeds up transmission, while still allowing large numbers to be represented when needed.

### Why unsigned? 

Range: By specifying "unsigned," it's clear that the integer can take values from 0 up to the maximum, utilizing the full 64 bits for the value itself. If the integer were signed, it would halve the range of positive values to accommodate negative numbers.

Application: Unsigned integers are often used in scenarios where negative numbers are not needed, such as counting, indexing, or representing large values where maximizing the range is important.

## Metadata Section

For example 

```
message CreateOrderRequest {
    int64 user_id = 1;
}
```

00001000

* Low first 3 low bits indicates the wire type -> 000
* First bit of the data section is called the most significant bit (MSB), and its
value is 0 when there is no additional byte. Its value becomes 1 if more bytes come to encode the remaining data.
* The remaining bits of the metadata section contain the field value. -> 0001 
