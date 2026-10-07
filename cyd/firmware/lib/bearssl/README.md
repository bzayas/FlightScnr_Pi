# BearSSL 0.6 (vendored)

[BearSSL](https://bearssl.org) by Thomas Pornin, MIT licence (`LICENSE.txt`).
`src/` and `inc/` are copied unmodified from `bearssl-0.6.tar.gz`
(SHA-256 `6705bba1714961b41a728dfc5debbe348d2966c117649392f8c8139efc83ff14`).

FlightScnr CYD uses it for HTTPS because it never allocates memory: the whole
TLS client (one 16.3 KB record buffer plus about 7 KB of state) is a fixed
static block, instead of the ~58 KB of heap that each request took with
mbedtls through WiFiClientSecure. See `src/net/fetch.cpp`.
