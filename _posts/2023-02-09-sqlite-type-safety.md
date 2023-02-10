# SQLite type safety

I love [SQLite][sqlite] but one of my biggest gripes with it is the dynamic type
system that it uses. To play devil's advocate and for the curious,
[here][flextypegood] is SQLite's article on the advantages. I have a personal
cheat sheet of ways to make SQLite feel less flexible and have decided to write
it up and share it. The biggest reason for this by far is that as of version
3.37.0 (2021-11-27) there are `STRICT` tables which handle a lot of this more
easily and I think it's now easy enough to do without much extra effort.

A good bit of this will just be summarizing [this][datatypes] documentation on
datatypes that SQLite has published but I will also be adding some tricks to be
more rigid. SQLite has *incredible* documentation and I highly recommend reading
it if you are curious about how SQL works in general. This article will not
assume you have read that document on datatypes but will assume that you will
open it if you are confused. I do not want to rehash their documentation here!

If you do not have SQLite installed then I suggest using [SQLiteStudio] which is
a great graphical tool for working with SQLite. It is super focused on SQLite
which does not provide any networking capabilities so the tool is super easy to
set up.

## Storage classes ("types")

These are the closest things you will get to types. They deal with how data is
stored on disk but that's not relevant here.

1. `NULL`: It is a null value. Nothing special.
2. `INTEGER`: Signed integers like 0, 1, 2, 3, -1, -2, -3, and so on.
3. `REAL`: Floating point values, sort of like decimals.
4. `TEXT`: Textual strings. Like `'Hello, World!'`.
5. `BLOB`: Raw binary data. It can be anything. It has no special conversions.

## Column affinity (column "type")

SQL has datatypes like `VARCHAR`, `INT`, and many others. When SQLite sees them
in table definitions it converts them to one of five "affinities". The specific
rules for determining are [here][affinity] for the curious. In my opinion you
should only use these five types directly and not rely on implicit conversions
in your DDL statements. The affinities are `TEXT`, `NUMERIC`, `INTEGER`, `REAL`,
and `BLOB`. (Also `ANY` is sort of an affinity but only with `STRICT` tables
which will be explained later.)

SQLite will try to convert data into the column's affinity. If it cannot then it
will leave it as is. That was a big factor in me looking for ways to fix this.
Without using `STRICT` tables you can end up with things like `'a'` in `INTEGER`
columns.

```sql
DROP TABLE IF EXISTS example1;
CREATE TABLE example1 (
    num INTEGER
);
INSERT INTO example1 (num) VALUES (1);
INSERT INTO example1 (num) VALUES ('a');
SELECT * FROM example1;
```

The results will be this. This is not what you would expect. Well, you probably
did because I told you it would happen. This is the sort of thing I want to
avoid.

| num |
| --- |
|   1 |
|   a |

I am being a little unfair here and showing you the bad aspects without showing
you the good. It does coerce types into other types when it is able to do so
losslessly. So the string `'2'` gets converted into the integer `2` when the
column's affinity is `INTEGER` because it can do so losslessly.

```sql
INSERT INTO example1 (num) VALUES ('2');
INSERT INTO example1 (num) VALUES ('3.0');
INSERT INTO example1 (num) VALUES ('4.1');
SELECT num, typeof(num) FROM example1;
```

| num | typeof(num) |
| --- | ----------- |
|   1 |     integer |
|   a |        text |
|   2 |     integer |
|   3 |     integer |
| 4.1 |        real |

### Conversion rules

All of the column affinities use the `NULL` and `BLOB` storage classes for
`NULL` and `BLOB` values. Those are always stored as-is. All of the attempted
conversions below are for the other types. *(`NULL` cannot go into columns with
the `NON NULL` constraint as normal.)*

1. `TEXT`: Converts `INTEGER` and `REAL` types to `TEXT`.
2. `NUMERIC`: There's a lot of nuance but basically it tries to store things as
   `INTEGER` first, then if unable to do so it tries `REAL`. Otherwise it stores
   as-is.
3. `INTEGER`: Confusingly, columns with this affinity behave the same as
   `NUMERIC` and only behave differently when used in `CAST` expressions.
4. `REAL`: Same as `NUMERIC` but converts `INTEGERS` into `REAL`.
5. `BLOB`: Stores everything as is.

## `STRICT` tables

Every column *must* have a datatype in `STRICT` tables. The datatypes you can
specify are `INT`, `INTEGER`, `REAL`, `TEXT`, `BLOB`, and `ANY`. `INT` and
`INTEGER` both become the `INTEGER` affinity.

Apart from the `ANY` datatype, the type of the value must be able to be
losslessly converted into the specified column's type or else an error is
raised. This is how most other SQL implementations work out of the box. `ANY`
just stores all types as they are in the table, much like `BLOB` in non-`STRICT`
tables. This allows you to have some flexibly typed columns if you want on
otherwise `STRICT` tables.

```sql
DROP TABLE IF EXISTS example2;
CREATE TABLE example2 (
    num INTEGER
) STRICT; -- This is how to make the table STRICT
INSERT INTO example2 (num) VALUES (1);
INSERT INTO example2 (num) VALUES ('a'); -- Errors
SELECT * FROM example2;
INSERT INTO example2 (num) VALUES ('2');
INSERT INTO example2 (num) VALUES ('3.0');
INSERT INTO example2 (num) VALUES ('4.1'); -- Errors
SELECT num, typeof(num) FROM example2;
```

| num | typeof(num) |
| --- | ----------- |
|   1 |     integer |
|   2 |     integer |
|   3 |     integer |

`ANY` in non-`STRICT` tables behaves as `NUMERIC`. See the previously mentioned
[affinity determiniation rules][affinity] for why. Because of that I would
suggest only using it in `STRICT` tables because that's the only place it has
special meaning.

## Other types, SQLite's "secret menu"

You may have seen or heard about people doing things like ordering items off of
the "secret menu" at places. These are things the kitchen is capable of making
but the store does not directly advertise. SQLite has a few of these lurking
under the hood. My criteria for these are things that it already has some
support for like booleans and various date/time formats but does not have types
for.

This will be done using `CHECK` constraints. There are a few different ways to
write them. SQLite actually reports the name of the `CHECK` constraint
violations (unlike `FOREIGN KEY` constraint violations) so I think naming them
is worthwhile. I do not have a good recommendation or strong opinion on how to
name them, but I think something is better than nothing. In the examples I may
leave them unnamed though.

```sql
CREATE TABLE demo (
    a INTEGER CHECK (a > 0), -- Unnamed in-line
    b INTEGER CONSTRAINT b_must_be_positive CHECK (b > 0), -- Named in-line
    c INTEGER,
    d INTEGER,
    CHECK (c > 0), -- Unnamed, not in-line
    CONSTRAINT d_must_be_positive CHECK (d > 0) -- Named, not in-line
) STRICT;
```

You may notice `IS` used a lot in the `CHECK` clauses. This is because `NULL`
does not equal `NULL`. `NULL = NULL` is false. Well, actually `NULL = NULL` is
`NULL`! The point is that if you do not use `IS` and use `=` instead then you
will not properly handle `NULL`s. If you want to reject them just use `NOT NULL`
like normal. (`foo INTEGER NOT NULL CHECK (...)`)

### Booleans

First, SQLite treats `TRUE` as the literal `1` and `FALSE` as the literal `0`.
Knowing that helps make the rest make more sense.

```sql
DROP TABLE IF EXISTS example3;
CREATE TABLE example3 (
    value INTEGER CONSTRAINT value_is_boolean CHECK (value IN (TRUE, FALSE))
) STRICT;
INSERT INTO example3 (value) VALUES (1);
INSERT INTO example3 (value) VALUES (0);
INSERT INTO example3 (value) VALUES (TRUE);
INSERT INTO example3 (value) VALUES (FALSE);
INSERT INTO example3 (value) VALUES (NULL);
SELECT * FROM example3;
INSERT INTO example3 (value) VALUES (2);
-- Last one shows the error messages,
-- CHECK constraint failed: value_is_boolean
```

| value |
| ----- |
|     1 |
|     0 |
|     1 |
|     0 |
|  NULL |

### Dates and times

There are a handful of date and time formats that SQLite can work with. You can
read about specifics [here][datetime] if you are curious how these formats and
functions work. There is some nuance there I am not going to cover here.

The goal is to accomplish a modest type checking using `CHECK` constraints like
we did in the above examples. The way this is done is by taking a value and
attempting to convert it into the desired format and then seeing if they match.

Some quick points about the date and time functions:

- All except for `strftime()` begin with the *time-value* and are followed by
  *modifiers*.
- `strftime()` is the same as those but has a *format* in front of *time-value*.
- The *time-value* is either a string representing various ISO 8601 formats (see
  below), a numeric value representing a Julian day number (see below), a
  numeric value representing a Unix timestamp (if it is followed by an `'auto'`
  or `'unixepoch'` modifier, still see below), or finally a string `'now'` which
  means right now.
- They output format matching the name of the function or in the case of
  `strftime()` matching the *format*.
- The ISO 8601 formats use UTC time by default but offsets can be provided.

#### [Julian days][julian]

I have never used this format personally but it may be useful for you. It is the
number of days (and fractional days) since noon in Greenwich on November 24th,
4714 BCE. It is represented by `REAL` values. Do not confuse this with the
Julian dates that accountants sometimes use which is the number of days since
the start of the year. The Unix epoch (midnight on January 1st, 1970 at UTC) is
`2440587.5` in this format. The `.5` on the end makes sense because it is at
midnight and measures from noon.

```sql
DROP TABLE IF EXISTS example4;
CREATE TABLE example4 (
    stamp REAL CONSTRAINT stamp_is_julian CHECK (stamp IS julianday(stamp))
); -- This table is not strict to demo that the check can prevent other formats
INSERT INTO example4 (stamp) VALUES (2440587.5);
INSERT INTO example4 (stamp) VALUES ('1970-01-01'); -- Violates the constraint
INSERT INTO example4 (stamp) VALUES (NULL);
SELECT stamp, datetime(stamp) FROM example4;
```

| stamp     | datetime(stamp)     |
| --------- | ------------------- |
| 2440587.5 | 1970-01-01 00:00:00 |
|      NULL |                NULL |

#### [ISO 8601][iso8601]

There are a variety of ISO 8601 formats SQLite can work with. I will present
four. The final is the trickiest so the example will use it. The first three
work the same as the Julian example.

1. `YYYY-MM-DD HH:MM:SS` with `datetime()`.
2. `YYYY-MM-DD` with `date()`.
3. `HH:MM:SS` with `time()`.
4. `YYYY-MM-DD HH:MM:SS.sss` with `strftime('%Y-%m-%d %H:%M:%f', ...)`. You can
   use any format that works with `strftime()` this way.

```sql
DROP TABLE IF EXISTS example5;
CREATE TABLE example5 (
    stamp TEXT CHECK (stamp IS strftime('%Y-%m-%d %H:%M:%f', stamp))
) STRICT;
INSERT INTO example5 (stamp) VALUES (NULL);
INSERT INTO example5 (stamp) VALUES ('1970-01-01 00:00:00'); -- Violates the constraint
INSERT INTO example5 (stamp) VALUES ('1970-01-01 00:00:00.000');
SELECT * FROM example5;
```

| stamp                   |
| ----------------------- |
|                    NULL |
| 1970-01-01 00:00:00.000 |

#### [Unix time][unixtime]

Unix timestamps are the number of seconds since midnight on January 1st, 1970 at
UTC represented by an `INTEGER`. Confusingly, when a numeric value is passed to
the date and time functions SQLite will treat it as a Julian day by default. To
correct this use the `'unixepoch'` modifier immediately following the
*time-value*. Also confusingly, this is the case when using the `unixepoch()`
function. It makes sense when you think about it but it looks odd at first.

```sql
DROP TABLE IF EXISTS example6;
CREATE TABLE example6 (
    stamp INTEGER CHECK (stamp IS unixepoch(stamp, 'unixepoch'))
) STRICT;
INSERT INTO example6 (stamp) VALUES (NULL);
INSERT INTO example6 (stamp) VALUES ('1970-01-01 00:00:00'); -- Violates the constraint
INSERT INTO example6 (stamp) VALUES (0);
SELECT stamp, datetime(stamp), datetime(stamp, 'unixepoch') FROM example6;
```

This demonstrates why using Unix timestamps is doable but slightly trickier. If
you forget the modifier you get incorrect values.

| stamp | datetime(stamp)      | datetime(stamp, 'unixepoch') |
| ----- | -------------------- | ---------------------------- |
|     0 | -4713-11-24 12:00:00 |          1970-01-01 00:00:00 |

### Enums

By now you should be an expert if you have read everything so I will keep this
example super short. You can use `IN (...)` to make enum types!

```sql
DROP TABLE IF EXISTS example7;
CREATE TABLE example7 (
    direction TEXT CHECK (direction IN ('LEFT', 'RIGHT'))
) STRICT;
INSERT INTO example7 (direction) VALUES ('UP'); -- Violates the constraint
INSERT INTO example7 (direction) VALUES (NULL); -- Is okay
```

## Closing thoughts

I hope this helps folks using SQLite and encourages people on the fence about it
due to the lack of a rich, static type system to give it a shot in personal
projects!

[sqlite]: https://sqlite.org/index.html
[flextypegood]: https://sqlite.org/flextypegood.html
[sqlitestudio]: https://www.sqlitestudio.pl/
[datatypes]: https://sqlite.org/datatype3.html
[affinity]: https://sqlite.org/datatype3.html#determination_of_column_affinity
[datetime]: https://sqlite.org/lang_datefunc.html
[julian]: https://en.wikipedia.org/wiki/Julian_day
[iso8601]: https://en.wikipedia.org/wiki/ISO_8601
[unixtime]: https://en.wikipedia.org/wiki/Unix_time

