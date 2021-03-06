# Huffman11

## Credits

- [IllegalArgument](https://github.com/IllegalArgument) for starting this project
- [Plato Mavropoulos](https://github.com/platomav) for invaluable help reconstructing the dictionaries and integrating this project with the [ME Analyzer](https://github.com/platomav/MEAnalyzer) firmware analysis tool
- The [Positive Research Team](https://github.com/ptresearch) for recovering the complete dictionaries and sharing their findings with the community in their [Intel ME 11.x Firmware Images Unpacker](https://github.com/ptresearch/unME11) tool

## About

This project is a reconstruction of the Huffman dictionaries used to compress version 11.x Intel ME modules. The dictionaries have been tested every version 11.x firmware image that I could find. For all modules in all of the tested versions, the hash of the decompressed output matches the hash in the corresponding metadata file. Note that the decompression script requires the decompressed size of the Huffman compressed module as an argument; this can be found in the corresponding metadata file for the module.

## Status

The dictionaries are complete enough to successfully decompress all version 11.x ME firmware that I was able to find. Thanks to the work of the Positive Research Team, the dictionaries should finally be complete with no unmapped codewords.

## Compression Details

Each Huffman-compressed module consists of a header section followed by streams of compressed data. The header section is made up of a variable number of 32-bit little-endian `LutEntry` structures formatted as `struct LutEntry { uint32_t addr:25; uint32_t flags:7; }`. Each `LutEntry` structure corresponds to a single stream of compressed data, with the `addr` field specifying the offset of the start of the compressed stream relative to the end of the header section and the `flags` field specifying which Huffman dictionary is needed to decompress the stream. Each compressed data stream decompresses to exactly `0x1000` bytes. Because of this, the number of `LutEntry` structures in the header (and therefore the size of the header as well) can be computed simply by dividing the decompressed size of the module by `0x1000`. This decompressed size can be found in the corresponding metadata (`.met`) file for every module.

The compression scheme used for the compressed data streams is form of prefix code that resembles a Huffman code. The main difference between the compression used by the ME and a classic Huffman code is that the symbols mapped to each codeword in the ME compression system do not all have the same length. I have found a few references to this sort of variable-length Huffman code in the literature, e.g., [Variable-Length Input Huffman Coding for System-on-a-Chip Test](https://eprints.soton.ac.uk/258321/1/pgonciari_tcad03.pdf) and [Improving Compression Ratio, Area Overhead, and Test Application Time for System-on-a-Chip Test Data Compression/Decompression](https://eprints.soton.ac.uk/256771/1/final-date02.pdf). The "industry" term for this type compression system seems to be "variable-length input Huffman coding" or "variable-length input Huffman compression", abbreviated as VIHC. The main applications discussed for VIHC tend to deal with compressing test data before sending it to automatic test equipment in the semiconductor fabrication process. Perhaps Intel is reusing compression hardware that is already present on the chip to decompress the ME modules, though this is purely speculation at this point.

In the version 11.x ME Huffman encoding, each codeword is between 7 and 15 bits long. The codewords used form a canonical Huffman code with the following "shape":

Codeword Length (in bits) | Highest Codeword (in binary) | Lowest Codeword (in binary)
------------------------- | ---------------------------- | ---------------------------
7 | `1111111` | `1110111`
8 | `11101101` | `10100011`
9 | `101000101` | `010111101`
10 | `0101111001` | `0011001011`
11 | `00110010101` | `00011011000`
12 | `000110101111` | `000011011111`
13 | `0000110111101` | `0000011001010`
14 | `00000110010011` | `00000001010000`
15 | `000000010011111` | `000000000000000`

Each codeword is mapped to a symbol that is between 1 and 15 bytes long. As mentioned above, there are two dictionaries that map these codewords to decompressed symbols. The value of the `flags` field in `LutEntry` structures seems to be either `0x40` or `0x60`; this value indicates which dictionary is used for the corresponding compressed stream. Both dictionaries use the same code "shape" in the table above, and, more interestingly, the same valued codewords in both dictionaries all seem to be mapped to symbols of the same length. For example, the codeword `11101100` in both dictionaries is mapped to a 15-byte symbol, `0000100011111` is mapped to an 8-byte symbol, etc.
