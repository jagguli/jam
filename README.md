# Overview
Jam (from an Indonesian word for "clock" or "time") is a time parsing
and processing library written to allow Riak's timeseries database to
use fine-grained date and time types.

Jam's initial release includes an ISO 8601 parser/formatter, but the
architecture allows for any syntax to be supported by adding a new
module that understands Jam's date/time structures.

There are several phases of processing available to allow an
application to massage the data for more sophisticated purposes than
merely translating strings to UNIX epoch time.

## Phases

### Parsing

Currently Jam supports (most of) ISO 8601 but other formats are
anticipated.

Yet to be supported:

* Date/time intervals
* Week dates

### Processing

The processing step converts the strings captured by parsing into a
(possibly valid) date and/or time. The resulting tuples will not be
the same as Erlang's date/time tuples because this library permits
fractional seconds, "incomplete" dates and times, and time zones.

### Validation

The processing step only exercises as much knowledge about "real"
times and dates as is necessary (e.g., knowing what years are leap
years to properly interpret ordinal dates). A time such as "25:15" is
a possible outcome of the parsing and processing steps, so validation
functions are supplied.

### Normalization

There are two unusual times which may arise.

The first, permitted by ISO 8601, is "24:00". This is *not* the same
as "00:00", at least when also attached to a date. "2016-06-15 24:00"
is the same as "2016-06-16 00:00" and normalization will convert the
former into the latter.

The second unusual time value: seconds > 59.

Occasionally [leap seconds](https://en.wikipedia.org/wiki/Leap_second)
will be added at the end of a day. UNIX/POSIX time will silently
["absorb"](https://en.wikipedia.org/wiki/Unix_time#Leap_seconds) that
into the first second of the following day.

Thus, a second value of `60` is considered valid by the validation
functions and is converted to `00` seconds in the following minute
during normalization.

### Translation

Dates and times can be converted to alternative time zones, to ISO
8601 (and, in the future, other formats), to Erlang's date/time
tuples, or to integers (UNIX epoch with support for sub-second
values).

## Illustrated usage


```erlang
1> jam_iso8601:parse("1985-04-23 13:15:57Z").
{parsed_datetime,{parsed_calendar,"1985","04","23"},
                 {parsed_time,"13","15","57",undefined,undefined,
                              {parsed_timezone,"Z",undefined,undefined}}}
2> jam:process(v(-1)).
{datetime,{date,1985,4,23},
          {time,13,15,57,undefined,undefined,{timezone,"Z",0,0}}}
3> jam:normalize(v(-1)).
{datetime,{date,1985,4,23},
          {time,13,15,57,undefined,undefined,{timezone,"Z",0,0}}}
4> jam:to_epoch(v(-1)).
483110157
5> jam_iso8601:to_string(v(-2)).
"1985-04-23T13:15:57Z"
6> jam_iso8601:to_string(v(-3), [{format, basic}]).
"19850423T131557Z"
7> jam_iso8601:to_string(v(-4), [{format, basic}, {z, false}]).
"19850423T131557+0000"
```

For a more sophisticated example, we handle a leap second with a
fractional component and time zone conversion.

Note that in order to support passing times without dates as
arguments, `jam:round_fractional_seconds/1` and `jam:convert_tz`
return a two-tuple, with the first value an integer expressing whether
or not a date adjustment resulted. Rounding the fractional second in
the example below does not change the date, but the time zone
conversion does.

```erlang
8> {_, DT} = jam:round_fractional_seconds(jam:normalize(jam:process(jam_iso8601:parse("20150630T23:59:60.738")))).
{0,
 {datetime,{date,2015,7,1},
           {time,0,0,1,undefined,undefined,undefined}}}
9> {_, DT} = jam:round_fractional_seconds(jam:normalize(jam:process(jam_iso8601:parse("20150630T23:59:60.738+00")))).
{0,
 {datetime,{date,2015,7,1},
           {time,0,0,1,undefined,undefined,{timezone,"+00",0,0}}}}
10> {_, NewDT} = jam:convert_tz(DT, "-07:30").
{-1,
 {datetime,{date,2015,6,30},
           {time,16,30,1,undefined,undefined,
                 {timezone,"-07:30",7,30}}}}
11> jam_iso8601:to_string(NewDT).
"2015-06-30T16:30:01-07:30"
```

Dates or times can be managed independently. This snippet also
illustrates compiling the ISO 8601 regular expressions for better
parsing performance.

```erlang
1> REs = jam_iso8601:init().
[{time,{re_pattern,14,0,
                   <<69,82,67,80,152,1,0,0,16,0,0,0,1,0,0,0,14,0,3,0,0,0,
                     ...>>}},
 {ordinal_date,{re_pattern,3,0,
                           <<69,82,67,80,125,0,0,0,16,0,0,0,1,0,0,0,3,0,0,0,0,...>>}},
 {calendar_date,{re_pattern,6,0,
                            <<69,82,67,80,172,0,0,0,16,0,0,0,1,0,0,0,6,0,3,0,...>>}},
 {week_date,{re_pattern,7,0,
                        <<69,82,67,80,174,0,0,0,16,0,0,0,5,0,0,0,7,0,3,...>>}},
 {ordinal_datetime,{re_pattern,17,0,
                               <<69,82,67,80,15,2,0,0,16,0,0,0,1,0,0,0,17,0,...>>}},
 {week_datetime,{re_pattern,21,0,
                            <<69,82,67,80,69,2,0,0,16,0,0,0,5,0,0,0,21,...>>}},
 {calendar_datetime,{re_pattern,20,0,
                                <<69,82,67,80,62,2,0,0,16,0,0,0,1,0,0,0,...>>}},
 {timezone,{re_pattern,5,0,
                       <<69,82,67,80,177,0,0,0,16,0,0,0,1,0,0,...>>}}]
2> Time = jam_iso8601:parse(REs, "T14:15:16-05").
{parsed_time,"14","15","16",undefined,undefined,
             {parsed_timezone,"-05","-05",undefined}}
```

The `iso8601_tests` module further illustrates usage of the `jam`
library.
