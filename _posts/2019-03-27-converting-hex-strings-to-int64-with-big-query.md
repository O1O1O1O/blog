---
layout: post
title: Converting a 64-bit hex string to INT64 with BigQuery
---

I recently had the need to convert a 64-bit hex string to an INT64 value in a BigQuery
user defined function (UDF).  I'll post more about why later but for now lets just say
you have a sixteen character hex string such as `7FFFFFFFFFFFFFFF` and need it as an INT64
value.  Easy you say, just prepend "0x" and do `CAST("0x7FFFFFFFFFFFFFFF" AS INT64)`.  
But what if you have something just a bit bigger, like `8000000000000000`?  Let's try that:

`SELECT CAST("0x8000000000000000" AS INT64);`
and we get the error:

`Could not cast literal "0x8000000000000000" to type INT64`

The problem is that [INT64]([https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#integer-type]) is a signed value in BigQuery (and other Google cloud databases like
Spanner) with values ranging from `-9,223,372,036,854,775,808` to `9,223,372,036,854,775,807`.

So we have to convert from the hex using [two's compliment](https://en.wikipedia.org/wiki/Two%27s_complement). 
I recently wrote some code in Python to do this:
```
def uint64_to_signedInt64(v):
    MAX_INT64 = 2**63-1

    if v > MAX_INT64:
        return (v & MAX_INT64) - (MAX_INT64 + 1)
    else:
        return v
```
But since we can't create an INT64 from the hex string we break it into two 32-bit values.  The most
significant is the first 8 characters of the hex string (remember each character in hex is 4-bits or a "nibble"). 
The least significant is the last 8 characters.  `v & MAX_INT64` is emulated by taking the most significant 32-bits
and `AND`-ing with `0x7fffffff` and shifting to the left 32 times then `OR`-ing with the least significant 32-bits.
This gives us a pure-SQL UDF like so:
```
CREATE TEMP FUNCTION
  hex64ToInt64(hex STRING)
  RETURNS INT64 AS (
    IF(hex < "8000000000000000", cast(concat("0x", hex) AS INT64), (
    SELECT (((ms32 & 0x7fffffff) << 32) | ls32) - 0x7fffffffffffffff - 1
    FROM (
      SELECT cast(concat("0x", substr(hex, 1, 8)) AS INT64) AS ms32,
             cast(concat("0x", substr(hex, 9, 8)) AS INT64) AS ls32))));
```

Now we can calculate `select hex64ToInt64("8000000000000000")` which is the most negative signed INT64 number `-9223372036854775808`
and `select hex64ToInt64("ffffffffffffffff")` is the least negative number `-1`.

I'll soon show you how I've used this function as part of a BigQuery emulation of Postgres' [HLL extension from Citus](https://github.com/citusdata/postgresql-hll)
