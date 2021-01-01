# Bluestone Industries Backup and Archive Tape (BI-BAT) format

This document outlines the format of the Bluestone Industries Backup and Archive Tape (BI-BAT), a storage medium for use with the Minecraft mods _OpenComputers_ and _Computronics_. _Note: The highest-capacity tape from Computronics (128 minutes, crafted with **Nether stars**) can store 31,457,280 bytes of data._

## Data types

These 4 data types are from [the TEC Redshift disk specification](https://github.com/Rami-Sabbagh/TEC-Redshift-Disk-Specification):

* Byte: 1 byte read from the tape, use as-is
* Boolean: 1 byte, True = 1, False = 0
* Int: 32-bit (4-byte) little-endian integer
* String:
    * (Int) Number of characters
    * (Byte[]) Non-terminated ASCII string

These 3 data types were created for BI-BAT, but may be used elsewhere:

* UUID: 128-bit (32-byte) big-endian integer
* BCD_Data: 1 byte, containing 2 BCD digits, big-endian
* Date_Time: (7 BCD_Data objects; 14 digits)
    * (4 digits) Year (i.e. 1999)
    * (2 digits) Month (i.e. 12 - December)
    * (2 digits) Day (i.e. 31)
    * (2 digits) Hour (i.e. 23 - 11 PM)
    * (2 digits) Minute (i.e. 59)
    * (2 digits) Second (i.e. 59)  

## Top-level data structure

* (String) The name of the archive
* (Date_Time) Date and time archive was created
* (UUID) Address of computer that created archive
* (Int) Number of sessions
* (Session[]) File/drive sessions, no separators

## Session data structure

* (Int) Size of remaining session data, in bytes
* (String) File name or drive label
* (Byte) Data source:
  * `0x00` = Undefined
  * `0x01` = File
  * `0x02` = Unmanaged hard drive
  * `0x03` = Unmanaged floppy disk
  * `0x04` = EEPROM
  * `0xF0-0xFF` = Test data
  * More codes can be added (application-specific)
* (Boolean) Is this session encrypted?
* (String) Password hint (Optional - use empty string if no hint)
* (Byte) Pre-compression delta-coding configuration
* (Byte) Post-compression delta-coding configuration
* (Int) Number of chunks
* (Chunk[]) Data chunks, no separators

Session data is split up into 'chunks' such that the **uncompressed data** of each chunk is 8,192 bytes in length.

## Chunk data structure

* (Int) Length of compressed data, in bytes
* (Int) CRC-32 checksum of compressed data
* (Byte[]) Compressed data

### Compression method

The specific implementation of delta coding used for this storage medium works on 2 stages; bytewise followed by bitwise for the encoding process, bitwise followed by bytewise for the decoding process. The "configuration byte" for each obfuscation stage represents the size of each "block" in bytes.

Obfuscation cycle:
* Next input block
* Output block = Input block XOR Buffer block
* Buffer block = Input block if encoding, Output block if decoding

The encoding process for a given chunk is shown below, reverse the process to decode:

* If encryption is enabled:
  * Encrypt data (AES, using hash as key)
* Repeat with all possible configurations, for smallest size:
  * If pre-compression delta coding is enabled:
    * Run delta coding (Pre-compression block size)
  * Compress data (Zlib / DEFLATE)
  * If post-compression delta coding is enabled:
    * Run delta coding (Post-compression block size)
* Generate CRC-32 checksum of compressed data
* Assemble compressed data and checksum into chunk

## Conclusion

This format is still a work-in-progress, which is why there aren't example programs to use it yet. If I have made **any** errors at all, or if you have any ideas for changes I should make to this format before making demo software, such as new data types, new features, or even *removing things*, feel free to send a feature request.
