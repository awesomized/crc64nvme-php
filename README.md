# crc-fast-php
[![Code Standards](https://github.com/awesomized/crc-fast-php/actions/workflows/code-standards.yml/badge.svg?branch=main)](https://github.com/awesomized/crc-fast-php/actions/workflows/code-standards.yml)
[![Static Analysis](https://github.com/awesomized/crc-fast-php/actions/workflows/static-analysis.yml/badge.svg?branch=main)](https://github.com/awesomized/crc-fast-php/actions/workflows/static-analysis.yml)
[![Unit Tests](https://github.com/awesomized/crc-fast-php/actions/workflows/unit-tests.yml/badge.svg?branch=main)](https://github.com/awesomized/crc-fast-php/actions/workflows/unit-tests.yml)
[![Latest Stable Version](http://poser.pugx.org/awesomized/crc-fast/v)](https://packagist.org/packages/awesomized/crc-fast)

Fast, SIMD-accelerated CRC computation in PHP via FFI using Rust. Currently supports [CRC-64/NVME](https://reveng.sourceforge.io/crc-catalogue/all.htm#crc.cat.crc-64-nvme) and [CRC-32/ISO-HDLC aka "crc32"](https://reveng.sourceforge.io/crc-catalogue/all.htm#crc.cat.crc-32-iso-hdlc). Other implementations welcome via PR.

## CRC-64/NVME 
Uses the [crc64fast-nvme](https://github.com/awesomized/crc64fast-nvme) Rust package and its C-compatible shared library.

It's capable of generating checksums at >20-50 GiB/s, depending on the CPU. It is much, much faster (>100X) than the native [crc32](https://www.php.net/manual/en/function.crc32.php), crc32b, and crc32c [implementations](https://www.php.net/manual/en/function.hash-algos.php) in PHP.

[CRC-64/NVME](https://reveng.sourceforge.io/crc-catalogue/all.htm#crc.cat.crc-64-nvme) is in use in a variety of large-scale and mission-critical systems, software, and hardware, such as:
- AWS S3's [recommended checksum](https://docs.aws.amazon.com/AmazonS3/latest/userguide/checking-object-integrity.html)
- The [Linux kernel](https://github.com/torvalds/linux/blob/786c8248dbd33a5a7a07f7c6e55a7bfc68d2ca48/lib/crc64.c#L66-L73)
- The [NVMe specification](https://nvmexpress.org/wp-content/uploads/NVM-Express-NVM-Command-Set-Specification-1.0d-2023.12.28-Ratified.pdf)

## CRC-32/ISO-HDLC (aka "crc32")
Uses the [crc32fast-lib]() Rust package (which exposes the [crc32fast](https://github.com/srijs/rust-crc32fast) Rust library as a C-compatible shared library).

It's >10X faster than PHP's native [crc32](https://www.php.net/manual/en/function.crc32.php) implementation.

[CRC-32/ISO-HDLC](https://reveng.sourceforge.io/crc-catalogue/all.htm#crc.cat.crc-32-iso-hdlc) is the de-facto "crc32" checksum, though there are [many other 32-bit variants](https://reveng.sourceforge.io/crc-catalogue/all.htm#crc.cat-bits.32).

## Changes

See the [change log](CHANGELOG.md).

## Requirements

You'll need to have built and installed the [crc64fast-nvme](https://github.com/awesomized/crc64fast-nvme) shared Rust library, and possibly configured where and how to load it. (See [Usage](#Usage), below).

## Installation

Use [Composer](https://getcomposer.org) to install this library (note the [Requirements](#Requirements) above):

```bash
composer require awesomized/crc-fast
```


## Usage

Examples are for `CRC-64/NVME`, but `CRC-32/ISO-HDLC` is nearly identical, just in a different namespace (`Awesomized\Checksums\Crc32\IsoHdlc`).

### Creating the CRC-64/NVME FFI object 

A [helper FFI Class](src/Ffi.php) is provided, which supplies many ways to easily create an FFI object for the [crc64fast-nvme](https://github.com/awesomized/crc64fast-nvme) shared library:

#### - Via [preloaded](https://www.php.net/manual/en/ffi.examples-complete.php) shared library (recommended for any long-running workloads, such as web requests):

```php
use Awesomized\Checksums\Crc64\Nvme;

// uses the opcache preloaded shared library and PHP Class(es)
$crc64Fast = Nvme\Ffi::fromPreloadedScope(
    scope: 'CRC64NVME', // optional, this is the default
);
```

#### - Via a C header file:
Uses a C header file to define the functions and point to the shared library (`.so` on Linux, `.dll` on Windows, `.dylib` on macOS, etc).

```php
use Awesomized\Checksums\Crc64\Nvme;

// uses the FFI_LIB and FFI_SCOPE definitions in the header file
$crc64Fast = Nvme\Ffi::fromHeaderFile(
    headerFile: 'path/to/crc64fast_nvme.h', // optional, can likely be inferred from the OS
);
```

#### - Via C definitions + library:

```php
use Awesomized\Checksums\Crc64\Nvme;

// uses the supplied C definitions and name/location of the shared library
$crc64Fast = Nvme\Ffi::fromCode(
    code: 'typedef struct DigestHandle DigestHandle;
            DigestHandle* digest_new(void);
            void digest_write(DigestHandle* handle, const char* data, size_t len);
            uint64_t digest_sum64(const DigestHandle* handle);
            void digest_free(DigestHandle* handle);',
    library: 'libcrc64fast_nvme.so',
);
```
### Using the CRC-64/NVME FFI object

#### Calculate CRC-64/NVME checksums:

```php
use Awesomized\Checksums\Crc64\Nvme;

/** @var \FFI $crc64Fast */

// calculate the checksum of a string
$checksum = Nvme\Computer::calculate(
    ffi: $crc64Fast, 
    string: 'hello, world!'
); // f8046e40c403f1d0

// calculate the checksum of a file, which will chunk through the file optimally,
// limiting RAM usage and maximizing throughput
$checksum = Nvme\Computer::calculateFile(
    ffi: $crc64Fast, 
    filename: 'path/to/hello-world'
); // f8046e40c403f1d0
```

#### Calculate CRC-64/NVME checksums with a Digest for intermittent / streaming / etc workloads:

```php
use Awesomized\Checksums\Crc64\Nvme;

/** @var \FFI $crc64FastNvme */

$crc64Digest = new Nvme\Computer(
    crc64Nvme: $crc64FastNvme,
);

// write some data to the digest
$crc64Digest->write('hello,');

// write some more data to the digest
$crc64Digest->write(' world!');

// calculate the entire digest
$checksum = $crc64Digest->sum(); // f8046e40c403f1d0
```

## Examples

There's a sample [CLI script](cli/calculate.php) that demonstrates how to use this library, or quickly calculate some checksums

## Development

This project uses [SemVer](https://semver.org), and has extensive coding standards, static analysis, and test coverage tooling. See the [Makefile](Makefile) for details.

Examples:

#### Building the shared `crc64fast-nvme` Rust library for local development and testing
```bash
make build
``` 
#### Validating PHP code
```bash
make validate
``` 

#### Repairing PHP code quality issues
```bash
make repair
```

Pull requests for improvements welcome.

## Testing

There's a [test suite](tests/) with `unit` test coverage.

#### Running the tests

```bash
make test
```
