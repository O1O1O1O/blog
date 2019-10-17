---
layout: post
title: 64-bit hash for Postgres and BigQuery
---

Did you ever Google for a solution to something, find one, then wonder who wrote it only to discover it was you
?  Well as of today I have. There had to be a reason the code looked so familiar and it fit my problem so perfectly I
 wasn't even paying attention to what website I was looking at that.  

Which brings me to a follow post to [64-bit hex number conversion](https://blog.0101010.com/converting-hex-strings-to-int64-with-big-query/) which describes how to convert a 16 character hex string to the BigQuery INT64 type.

## Why 64-bit numbers gain?

Today I found myself needing a 64-bit hash function to generate something I can use as an identity for some
potentially very large string columns.  Previously I had been using the BigQuery function [FARM_FINGERPRINT](https://cloud.google.com/bigquery/docs/reference/standard-sql/hash_functions) which
Google promises is both very fast and a very good hash although it isn't designed to be cryptographically strong
and is only 64-bits.  However since I'm dealing with tens, if not hundreds of millions of strings the chances of a
random collision with 64-bits is still vanishingly small (one in billions).
    
## The problem with Farm fingerprint

The problem with Farm fingerprint is although it is based on the open source [FarmHash](https://github.com/google/farmhash) 
library it doesn't seem to be widely used elsewhere.  Initially I was only using the hash values internally to
normalize some tables and avoid repeating big long strings.  The data was being computed in BigQuery and exported to
Postgres and the later didn't really need to know what hash algorithm was being used.

## Postgres' best alternative?

Since I found myself wanting to also generate these hash values in Postgres but there are no implementations of
FarmHash for Postgres that I could use. Looking at the [list of hash functions](https://www.postgresql.org/docs/11/pgcrypto.html) 
we find that MD5 is by far the fastest with 150M hashes/sec on their benchmark system which is 4x the nearest
competing hash SHA1.

## 64-bit MD5 in Postgres

For maximum compactness I'd like the id to be just 64-bits which as mentioned above is I believe sufficiently
unlikely to cause a collision for my data set that I don't care.  After all Google seems happy with 64-bits for
their farm fingerprint hash which gives a total of 16 billion billion combinations (that's 16 quintillion according
to [Wikipedia](https://en.wikipedia.org/wiki/Names_of_large_numbers)).  The MD5 function on Postgres outputs a 128-bit
value in the form of a hex string.  

So if you do:
```sql
select md5('hello, world');
```
you'll get the string `e4d7f1b4ed2e42d15898f4b27b019da4`.

Now it is relatively straightforward to just chop off the first 16 characters which represent the most significant
64-bits of the value and convert to a 64-bit int. A
StackOverflow [post](https://stackoverflow.com/a/9812029) offers up this pure SQL function to do that with the
minimum of effort for a string input:

```sql
create function h_bigint(text) returns bigint as $$
 select ('x'||substr(md5($1),1,16))::bit(64)::bigint;
$$ language sql;
```

But this left me wondering - how do you do this with BigQuery?

## MD5 on BigQuery

Like Postgres, BigQuery offers MD5 as a builtin. But the return value of this function is of type `BYTES` and 
length 16.  So our
original hello world test now gives us this result `5NfxtO0uQtFYmPSyewGdpA==` which is a Base64 encoding of the
128-bit MD5 value.  

Now if you look at the [conversion rules](https://cloud.google.com/bigquery/docs/reference/standard-sql/conversion_rules) 
for BigQuery you will find that the only things you can convert `BYTES` to are more `BYTES` and `STRING`.

But it turns out that BigQuery has a handy-dandy [`to_hex()`](https://cloud.google.com/bigquery/docs/reference/standard-sql/functions-and-operators#to_hex)
function that takes bytes and outputs the hexadecimal version.  So now if we do:
```sql
select to_hex(md5('hello, world'));
```
we get the hex string `e4d7f1b4ed2e42d15898f4b27b019da4` which fortunately is exactly what Postgres gave us before.
A simple application of `substr()` will also give us the most-significant 64-bits which we can then cast to an `int64`
type.   Let's try that...
```sqlite-psql
select cast(substr(to_hex(md5('hello, world')), 1, 16) as int64);
```
And BOOM! there it is: `Bad int64 value: e4d7f1b4ed2e42d1`

Bugger.

## Past me helps future me

So this is where I Googled ["converting 64 bit hex string to int64 on BigQuery"](https://www.google.com/search?q=converting+64+bit+hex+string+to+int64+on+BigQuery) 
and found myself brought [right back here](https://blog.0101010.com/converting-hex-strings-to-int64-with-big-query/) as
previously mentioned.

With help from past me I can now define a temporary user defined function (UDF) in SQL that calculates the 16 character
hex string of the first 64-bits of the MD5 and then calls my previously defined 64-bit hex to `int64` conversion
UDF like so:
```sql
CREATE TEMP FUNCTION
  hex64_to_int64(hex STRING)
  RETURNS INT64 AS (
    IF(hex < "8000000000000000", 
       cast(concat("0x", hex) AS INT64), 
       (SELECT (((ms32 & 0x7fffffff) << 32) | ls32) - 0x7fffffffffffffff - 1
        FROM (SELECT cast(concat("0x", substr(hex, 1, 8)) AS INT64) AS ms32,
                     cast(concat("0x", substr(hex, 9, 8)) AS INT64) AS ls32))));
CREATE TEMP FUNCTION
  md5_int64(s STRING)
  RETURNS INT64 AS (
    hex64_to_int64(substr(to_hex(md5(s)),1,16))
  );
```  
Now I can simply call `select md5_int64("hello, world");` and get back `-1956829753693551919` which is 
exactly what the Postgres `h_bigint` function returns (phew!).
