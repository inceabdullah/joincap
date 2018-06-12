# joincap

Merge multiple pcap files together, gracefully.

[![CircleCI](https://circleci.com/gh/assafmo/joincap.svg?style=shield&circle-token=cd4f46d248b7601530558ae6559a20ff75a897ad)](https://circleci.com/gh/assafmo/joincap)
[![Go Report Card](https://goreportcard.com/badge/github.com/assafmo/joincap)](https://goreportcard.com/report/github.com/assafmo/joincap)

## Why?

I believe skipping corrupt packets is better than failing the entire merge job.  
When using `tcpslice` or `mergecap` sometimes `pcapfix` is needed to fix bad input pcap files.

1.  One option is to try and run merge (`mergecap`/`tcpslice`), if we get errors then run `pcapfix` on the bad pcaps and then run merge again.
    - Adds complexity (run -> check errors -> fix -> rerun)
    - (If errors) Demands more resources (`pcapfix` processes)
    - (If errors) Extends the total run time
2.  Another option is to run `pcapfix` on the input pcap files and then merge.
    - Extends the total run time by a lot (read and write each pcap twice instead of once)
    - Demands more storage (for the fixed pcaps)
    - Demands more resources (`pcapfix` processes)
3.  We can use `pcapfix` "in memory" with process substitution: `mergecap -w out.pcap <(pcapfix -o /dev/stdout 1.pcap) <(pcapfix -o /dev/stdout 2.pcap)`.
    - Adds complexity (build a complex command line)
    - Demands more resources (`pcapfix` processes)
    - Harder for us to use pathname expansion (e.g. `tcpslice -w out.pcap *.pcap`)
    - We have to mind the command line character limit (in case of long pathnames)
    - Doesn't work for `tcpslice` (seeks the last packets to calculate time ranges - cannot do this with pipes)

## Error handling: `tcpslice` vs `mergecap` vs `joincap`

| Use case                                                                                           | joincap                                                                 | tcpslice v1.2a3                                                                                                                                                       | mergecap v2.4.5                                                                                                                                                                   | Example                                                                                                           |
| -------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Corrupt input global header                                                                        | :heavy_check_mark: Ignores the corrupt input pcap                       | :x: Dies with `tcpslice: bad tcpdump file pcap_examples/bad_global.pcap: archaic pcap savefile format`                                                                | :x: Dies with `mergecap: The file "pcap_examples/bad_global.pcap" contains record data that mergecap doesn't support. (pcap: major version 0 unsupported)`                        | Merge `pcap_examples/bad_global.pcap`                                                                             |
| Corrupt input packet header                                                                        | :heavy_check_mark: Ignores the packet and tries to find the next header | :x: Infinite loop?                                                                                                                                                    | :x: Dies with `mergecap: The file "pcap_examples/bad_first_header.pcap" appears to be damaged or corrupt. (pcap: File has 2368110654-byte packet, bigger than maximum of 262144)` | Merge `pcap_examples/bad_first_header.pcap`                                                                       |
| Unexpectd EOF (last packet data is truncated)                                                      | :heavy_check_mark: Ignores the last packet                              | :heavy_check_mark: Pads the last packet                                                                                                                               | :heavy_check_mark: Pads the last packet                                                                                                                                           | Merge `pcap_examples/unexpected_eof_on_first_packet.pcap` or `pcap_examples/unexpected_eof_on_second_packet.pcap` |
| Input pcap has no packets (global header is ok, no first packet header)                            | :heavy_check_mark: Ignores the empty input pcap                         | :x: Outputs empty pcap (Only global header)                                                                                                                           | :heavy_check_mark: Ignores the empty pcap                                                                                                                                         | Merge `pcap_examples/ok.pcap` with `pcap_examples/no_packets.pcap`                                                |
| Input file size is smaller than 24 bytes (global header is truncated)                              | :heavy_check_mark: Ignores the corrupt input pcap                       | :x: Dies with `tcpslice: bad tcpdump file pcap_examples/empty: truncated dump file; tried to read 4 file header bytes, only got 0`                                    | :heavy_check_mark: Ignores the corrupt pcap                                                                                                                                       | Merge `pcap_examples/ok.pcap` with `pcap_examples/empty` or `pcap_examples/partial_global_header.pcap`            |
| Input file size is between 24 and 40 bytes (global header is ok, first packet header is truncated) | :heavy_check_mark: Ignores the corrupt input pcap                       | :x: Dies with `tcpslice: bad status reading first packet in pcap_examples/partial_first_header.pcap: truncated dump file; tried to read 16 header bytes, only got 11` | :x: Dies with `mergecap: The file "pcap_examples/partial_first_header.pcap" appears to have been cut short in the middle of a packet.`                                            | Merge `pcap_examples/ok.pcap` with `pcap_examples/partial_first_header.pcap`                                      |
| Input file doesn't exists                                                                          | :heavy_check_mark: Ignores the non existing input file                  | :x: Dies with `tcpslice: bad tcpdump file ./not_here: ./not_here: No such file or directory`                                                                          | :x: Dies with `mergecap: The file "./not_here" doesn't exist.`                                                                                                                    | Merge `pcap_examples/ok.pcap` with `./not_here`                                                                   |
| Input file is a directory                                                                          | :heavy_check_mark: Ignores the non existing input file                  | :x: Dies with `tcpslice: bad tcpdump file examples: error reading dump file: Is a directory`                                                                          | :x: Dies with `mergecap: "examples" is a directory (folder), not a file.`                                                                                                         | Merge `pcap_examples/ok.pcap` with `pcap_examples/`                                                               |
| Input file end is garbage                                                                          | :heavy_check_mark: Ignores the corrupt end of the pcap                  | :x: Dies with `tcpslice: problems finding end packet of file pcap_examples/bad_end.pcap`                                                                              | :heavy_check_mark: Ignores the corrupt end of the pcap                                                                                                                            | Merge `pcap_examples/ok.pcap` with `pcap_examples/bad_end.pcap`                                                   |
| Input file is gzipped (.pcap.gz)                                                                   | :heavy_check_mark: Works                                                | :x: Dies with `tcpslice: bad tcpdump file pcap_examples/ok.pcap.gz: unknown file format`                                                                              | :heavy_check_mark: Works                                                                                                                                                          | Merge `pcap_examples/ok.pcap.gz`                                                                                  |

## Install

```bash
go get -u github.com/assafmo/joincap
```

## Usage

```bash
Usage:
  joincap [OPTIONS] InFiles...

Application Options:
  -v, --verbose  Explain when skipping packets or entire input files.
  -V, --version  Print the version and exit.
  -w=            Sets the output filename. If the name is '-', stdout will be used. (default: -)

Help Options:
  -h, --help     Show this help message
```

## Benchmarks

|              | Version | Speed    | Time     |
| ------------ | ------- | -------- | -------- |
| **mergecap** | 2.4.5   | 597MiB/s | 0m5.573s |
| **tcpslice** | 1.2a3   | 887MiB/s | 0m3.464s |
| **joincap**  | 0.8.3   | 439MiB/s | 0m6.995s |

- Merging 3 files with total size of 2.99994GiB.
- Running on Linux 4.15.0-22-generic, with Intel(R) Core(TM) i5-8250U CPU @ 1.60GHz (with SSE4.2), with 7873 MB of physical memory, with locale C, with zlib 1.2.11.
