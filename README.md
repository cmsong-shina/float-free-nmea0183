[![Crates.io](https://img.shields.io/badge/crates.io-v0.5.0-orange.svg?longCache=true)](https://crates.io/crates/nmea0183)
[![master](https://github.com/nsforth/nmea0183/actions/workflows/rust.yml/badge.svg)](https://github.com/nsforth/nmea0183/actions/workflows/rust.yml)
# NMEA 0183 parser.

This repository fork of nice [nsforth/nmea0183](https://github.com/nsforth/nmea0183/tree/master) project.

But on ch32v003, MCU which 16KB flash without FPU, it produces (relatively) huge binary so it can not be flashed.

```
File  .text    Size             Crate Name
0.2%  13.1%  3.0KiB         [Unknown] main
0.1%   5.8%  1.3KiB compiler_builtins compiler_builtins::float::div::div
0.1%   5.4%  1.2KiB compiler_builtins compiler_builtins::float::add::add
0.1%   4.9%  1.1KiB compiler_builtins compiler_builtins::float::mul::mul
0.1%   4.5%  1.0KiB              core core::num::dec2flt::<impl core::str::traits::FromStr for f64>::from_str
0.1%   4.3%   1008B              core core::num::dec2flt::<impl core::str::traits::FromStr for f32>::from_str
0.1%   3.6%    844B compiler_builtins compiler_builtins::float::div::div
0.1%   3.4%    804B              core core::num::dec2flt::decimal_seq::parse_decimal_seq
0.0%   3.1%    728B              core core::num::dec2flt::lemire::compute_float
0.0%   2.9%    692B              core core::num::dec2flt::lemire::compute_float
0.0%   2.9%    672B compiler_builtins compiler_builtins::int::specialized_div_rem::u64_div_rem
0.0%   2.8%    662B              core core::num::dec2flt::parse::parse_number
0.0%   2.5%    584B compiler_builtins compiler_builtins::float::mul::mul
0.0%   2.3%    540B              core core::num::dec2flt::decimal_seq::DecimalSeq::left_shift
0.0%   2.3%    538B              core core::num::dec2flt::lemire::compute_product_approx
0.0%   2.2%    510B              core core::num::dec2flt::decimal_seq::DecimalSeq::right_shift
0.0%   1.9%    452B              core core::num::dec2flt::parse::try_parse_digits
0.0%   1.9%    450B              core <core::str::iter::Split<P> as core::iter::traits::iterator::Iterator>::next
0.0%   1.7%    398B          nmea0183 nmea0183::coords::Latitude::parse
0.0%   1.7%    394B          nmea0183 nmea0183::coords::Longitude::parse
0.4%  26.2%  6.0KiB                   And 124 smaller methods. Use -n N to show more.
1.6% 100.0% 22.9KiB                   .text section size, the file size is 1.4MiB
```

So this library aims for binary size at the expense of functionality and accuracy. MCUs with flash space of 32KB or more or FPU have no reason to use this library.

Implemented most used sentences like RMC, VTG, GGA, GLL, GSV, GSA.
Parser do not use heap memory and relies only on `core`.

You should instantiate [Parser](https://docs.rs/nmea0183/latest/nmea0183/struct.Parser.html) with [new](https://docs.rs/nmea0183/latest/nmea0183/struct.Parser.html#method.new) and than use methods like [parse_from_byte](https://docs.rs/nmea0183/latest/nmea0183/struct.Parser.html#method.parse_from_bytes) or [parse_from_bytes](https://docs.rs/nmea0183/latest/nmea0183/struct.Parser.html#method.parse_from_bytes).
If parser accumulates enough data it will return [ParseResult](https://docs.rs/nmea0183/latest/nmea0183/enum.ParseResult.html) on success or `&str` that describing an error.

You do not need to do any preprocessing such as split data to strings or NMEA sentences.

# Examples

If you could read a one byte at a time from the receiver you may use `parse_from_byte`:
```rust
use nmea0183::{Parser, ParseResult};

let nmea = b"$GPGGA,145659.00,5956.695396,N,03022.454999,E,2,07,0.6,9.0,M,18.0,M,,*62\r\n$GPGGA,,,,,,,,,,,,,,*00\r\n";
let mut parser = Parser::new();
for b in &nmea[..] {
    if let Some(result) = parser.parse_from_byte(*b) {
        match result {
            Ok(ParseResult::GGA(Some(gga))) => { }, // Got GGA sentence
            Ok(ParseResult::GGA(None)) => { }, // Got GGA sentence without valid data, receiver ok but has no solution
            Ok(_) => {}, // Some other sentences..
            Err(e) => { } // Got parse error
        }
    }
}
```

If you read many bytes from receiver at once or want to parse NMEA log from text file you could use Iterator-style:
```rust
use nmea0183::{Parser, ParseResult};

let nmea = b"$GPGGA,,,,,,,,,,,,,,*00\r\n$GPRMC,125504.049,A,5542.2389,N,03741.6063,E,0.06,25.82,200906,,,A*56\r\n";
let mut parser = Parser::new();

for result in parser.parse_from_bytes(&nmea[..]) {
    match result {
        Ok(ParseResult::RMC(Some(rmc))) => { }, // Got RMC sentence
        Ok(ParseResult::GGA(None)) => { }, // Got GGA sentence without valid data, receiver ok but has no solution
        Ok(_) => {}, // Some other sentences..
        Err(e) => { } // Got parse error
    }
}
```

It is possible to ignore some sentences or sources. You can set filter on [Parser](https://docs.rs/nmea0183/latest/nmea0183/struct.Parser.html) like so:
```rust
use nmea0183::{Parser, ParseResult, Sentence, Source};

let parser_only_gps_gallileo = Parser::new()
    .source_filter(Source::GPS | Source::Gallileo);
let parser_only_rmc_gga_gps = Parser::new()
    .source_only(Source::GPS)
    .sentence_filter(Sentence::RMC | Sentence::GGA);
```

# Panics

Should not panic. If so please report issue on project page.

# Errors

`Unsupported sentence type.` - Got currently not supported sentence.

`Checksum error!` - Sentence has wrong checksum, possible data corruption.

`Source is not supported!` - Unknown source, new sattelite system is launched? :)

`NMEA format error!` - Possible data corruption. Parser drops all accumulated data and starts seek new sentences.

It's possible to got other very rare error messages that relates to protocol errors. Receivers nowadays mostly do not violate NMEA specs.
