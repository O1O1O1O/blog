---
layout: post
title: Making Postgres date_trunc() use a Sunday based week
---

Postgres provides a function called [`date_trunc()`](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-TRUNC) 
which rounds a timestamp (or date) to various granularities including:

- day
- week
- month
- quarter
- year

The value returned is the earliest moment of the specified interval that includes the supplied timestamp for example:
```sql
SELECT date_trunc('year', '2001-02-16'::date)::date;
```
gives the result `2001-01-01`.

## The problem with weeks...

The problem comes when you supply `week` as the granularity.  Postgres thinks a week starts on a Monday so:
```sql
SELECT date_trunc('week', '2019-09-17'::date)::date;
```
returns `2019-09-16` which is a Monday.

So why is that a problem?  Well not everyone thinks Monday is the start of the week.  At my company apparently our
customers usually think Sunday is the start of the week.  This [Quora response](https://www.quora.com/Does-the-week-start-on-Sunday-or-Monday-1/answer/Naveen-Krishna-15) sums it up:

> The first day of the week varies all over the world. In most  cultures, Sunday is regarded as the first day of the week although many  observe Monday as the first day of the week.
> 
> According to the Bible, the  Sabbath or Saturday is the last day of the week which marks Sunday as  the first day of the week for many Jewish and Christian faiths, while  many countries regard Monday as the first day of the week. 
> 
> According to the International standard ISO 8601, Monday is the first day of the week ending with Sunday  as the seventh day of the week. Although this is the international  standard, countries such as the United States still have their calendars  refer to Sunday as the start of the seven-day week.

Personally I'm all about standards over conventions so if there is an ISO standard I'll follow that, it also coincides
with my work week, never mind what some Biblical Sunday based convention says.

And yet the customers still want a Sunday based week.  

I looked and there is no hidden setting for Postgres that forces it to use a Sunday based week in `date_trunc()`.
Fortunately the solution here is actually very simple, just use relativity!  No, not Einstein's relativity, just
a relative calendar.  

Let's imagine there is your calendar where some day other than Monday is the start of the week, and there is the
relative calendar where `Monday` is the start of week.  Imagine you calendar is, as is
the most common alternative, one where `Sunday` is the start of week.  If we add `one day` to the date in your calendar then every `Sunday` 
would then be the next day i.e. `Monday`, this becomes the relative calendar date.  If we compute `date_trunc()` on 
that date it will still be the same `Monday` so
that's good. Similarly `Monday` in your calendar becomes `Tuesday` in the relative one and `date_trunc()` will return 
`Monday` before that Tuesday. Similarly every `Saturday` in your calendar becomes the `Sunday` after in the relative 
calendar so `date_trunc()` will return the `Monday` before it.  As we every `Monday` in the relative calendar is 
equivalent to the `Sunday before it` in your calendar.  This is exactly what we want!

Let's codify this process.

So if we think Sunday is the start of week add `one day` to the date before passing to `date_trunc()` which gives a
result for a relative calendar where `Monday` is equivalent to your `Sunday`.  So we then subtract `one day` from
the result of `date_trunc()` and get the start of week in your calendar.  

Conveniently in Postgres if you have a value that is a `date` you can simply add and subtract an integer and that's
the same as adding and subtracting the equivalent number of days. 

So:
```sql
SELECT date_trunc('week', '2019-09-17'::date + 1)::date - 1;
```
does the relative calendar calculation we want and returns `2019-09-15` which is Sunday.

What about the boundary conditions of a date that is already Sunday and is the last day of a week? 

```sql
-- Sunday 2019-09-15
SELECT date_trunc('week', '2019-09-15'::date + 1)::date - 1;
-- Saturday 2019-09-14
SELECT date_trunc('week', '2019-09-14'::date + 1)::date - 1;
```

return `2019-09-15` (Sunday) and `2019-09-08` (previous Sunday) which is exactly what we want.

Also note we could just as easily subtract `six days` and then add `six days` after the result.  They both work - it's all relative!
As you can see you could easily generalize this into a function `week_trunc()` that lets you say which day of the 
week should be considered the start and compute an appropriate offset to add and then subtract which is `number of days that Monday is 
ahead of your week start day`. Alternatively think of it like "if your week starts on day 0 and ends on day 6 what
day number is Monday in your week?".

Here's that function for you with a default of Monday based weeks:

```sql
CREATE FUNCTION week_trunc(date, integer DEFAULT 0) RETURNS DATE AS $$
 SELECT DATE_TRUNC('week', $1 + $2)::date - $2
$$ LANGUAGE SQL;
```