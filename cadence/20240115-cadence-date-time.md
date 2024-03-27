---
status: draft
flip: 245
authors: darkdrag00n (darkdrag00n@proton.me)
sponsor: Bastian Müller (bastian@dapperlabs.com)
updated: 2024-01-29
---

# FLIP 245: Date & Time

## Objective

This FLIP proposes adding Date & Time types and manipulation utilities to Cadence. It proposes two new types, `LocalDateTime` and `Duration` along with functions to create and manipulate them.

## Motivation

Manipulating Date & Time is a common aspect of user programs. Presently, Cadence users have to implement these calculations using raw `UFix64` timestamps.

Doing them manually is both hard and error-prone. As a result, standard library support similar to other modern languages would enhance the usability of the language.

## User Benefit

Manipulating date and time using raw timestamps is difficult and error-prone. Providing rich suite of standard library types and functions would increase the usability and user-experience.

## Design Proposal

New types `LocalDateTime` & `Duration` will be added to Cadence. Both of these data types assume that there are 3600*24 seconds in every day and there are no leap years.

### LocalDateTime

The `LocalDateTime` type will be defined as follows:

```cadence
pub struct LocalDateTime {
    let timestamp : UFix64
}
```

Please note the following points:
1. The stored timestamp represents the seconds elapsed since epoch (1st Jan 1970 00:00:00z)
2. The type will **not** be timezone aware i.e. it'll be the responsibility of the application developer to handle timezone details.
3. It is assumed that there are 3600*24 seconds in every day and there are no leap years.

Apart from the `LocalDateTime`, a few other types will be added to Cadence. They are:

1. `Month`
2. `Day`

```cadence
pub enum Month: UInt8 {
    pub case january
    pub case february
    pub case march
    pub case april
    pub case may
    pub case june
    pub case july
    pub case august
    pub case september
    pub case october
    pub case november
    pub case december
}

pub enum Day: UInt8 {
    pub case monday
    pub case tuesday
    pub case wednesday
    pub case thursday
    pub case friday
    pub case saturday
    pub case sunday
}
```

#### Constructor Functions
Constructor functions will be defined to instantiate variables of type `LocalDateTime`. They are defined below.

- `LocalDateTime.now()`: Creates a `LocalDateTime` with value based on the current block timestamp.

- `LocalDateTime.fromTimestamp(timestamp : UFix64)`: Creates a `LocalDateTime` with value based on the passed timestamp argument.

- `LocalDateTime.of(year: UInt64, month: Month, day: UInt8, hour: UInt8, minute: UInt8, second: UInt8)`: Creates a `LocalDateTime` with value based on the passed arguments assuming that there are 3600*24 seconds in every day.

#### Public Members
The `LocalDateTime` will provide the following public members:

1. `getYear(): UInt32`: Get the year of the date-time.
2. `getMonth(): Month`: Get the month of the date-time.
3. `getDate(): UInt8`: Get the date of the date-time.
4. `getHour(): UInt8`: Get the hour of the date-time.
5. `getMinute(): UInt8`: Get the minute of the date-time.
6. `getSecond(): UInt8`: Get the second of the date-time.
7. `getDayOfWeek(): Day`: Get the day-of-week of the date-time.

#### Examples
Example usage of `LocalDateTime` are:

```cadence
/// Assume that the current block timestamp is 1707045210
let currentDateTime = LocalDateTime.now()
currentDateTime.getYear()      // 2024
currentDateTime.getMonth()     // february
currentDateTime.getDate()      // 04
currentDateTime.getHour()      // 11
currentDateTime.getMinute()    // 13
currentDateTime.getSecond()    // 30
currentDateTime.getDayOfWeek() // sunday

let dateTimeFromTs = LocalDateTime.fromTimestamp(1415829132)
dateTimeFromTs.getYear()      // 2014
dateTimeFromTs.getMonth()     // november
dateTimeFromTs.getDate()      // 12
dateTimeFromTs.getHour()      // 21
dateTimeFromTs.getMinute()    // 52
dateTimeFromTs.getSecond()    // 12
dateTimeFromTs.getDayOfWeek() // wednesday

let dateTimeFromComponents = LocalDateTime.of(2019, april, 29, 19, 49, 31)
dateTimeFromComponents.getYear()      // 2019
dateTimeFromComponents.getMonth()     // april
dateTimeFromComponents.getDate()      // 29
dateTimeFromComponents.getHour()      // 19
dateTimeFromComponents.getMinute()    // 49
dateTimeFromComponents.getSecond()    // 31
dateTimeFromComponents.getDayOfWeek() // monday
```

### Duration
A `Duration` will represent the difference between two dates or times. It will defined as follows:

```cadence
pub struct Duration {
    let microseconds: UInt64
    let seconds: UInt64
    let days: Int64
}
```

The three internal fields will be stored in the normalized form with the following restrictions:
1. `0 <= microseconds < 10^6`
2. `0 <= seconds < 3600*24`
3. `-999999999 <= days <= 999999999`

This model is taken from the time-tested Python [datetime module](https://docs.python.org/3/library/datetime.html#timedelta-objects).

#### Constructor Functions
Constructor functions will be defined to instantiate variables of type `Duration`. They are defined below.

- `Duration.betweenLocalDateTime(t1 : LocalDateTime, t2 : LocalDateTime)`: Creates a `Duration` which represents the time duration between arguments `t1` and `t2`.

- `Duration.of(years: Int64, days: Int64, hours: Int64, minutes: Int64, seconds: Int64, miliseconds: Int64, microseconds: Int64)`: Creates a `Duration` with value based on the passed arguments assuming that there are 3600*24 seconds in every day.

During the normalization:
1. 1 year is converted to 365 days
2. 1 hour is converted to 3600 seconds
3. 1 minute is converted to 60 seconds
4. 1 milisecond is converted to 1000 microseconds

#### Public Members
The `Duration` will provide the following public members:

- `getDays(): Int32`: Get the days of the `Duration`.
- `getSeconds(): UInt32`: Get the second of the `Duration`.
- `getMicroseconds(): UInt32`: Get the microseconds of the `Duration`.
- `addYears(years : UInt32): Duration`: Return a new `Duration` after adding the provided number of years to it.
- `addDays(days : UInt32): Duration`: Return a new `Duration` after adding the provided number of days to it.
- `addHours(hours : UInt32): Duration`: Return a new `Duration` after adding the provided number of hours to it.
- `addMinutes(minutes : UInt32): Duration`: Return a new `Duration` after adding the provided number of minutes to it.
- `addSeconds(seconds : UInt32): Duration`: Return a new `Duration` after adding the provided number of seconds to it.
- `addMiliseconds(ms : UInt32): Duration`: Return a new `Duration` after adding the provided number of miliseconds to it.
- `addMicroseconds(microsecs : UInt32): Duration`: Return a new `Duration` after adding the provided number of microseconds to it.
- `subtractYears(years : UInt32): Duration`: Return a new `Duration` after subtracting the provided number of years from it.
- `subtractDays(days : UInt32): Duration`: Return a new `Duration` after subtracting the provided number of days from it.
- `subtractHours(hours : UInt32): Duration`: Return a new `Duration` after subtracting the provided number of hours from it.
- `subtractMinutes(minutes : UInt32): Duration`: Return a new `Duration` after subtracting the provided number of minutes from it.
- `subtractSeconds(seconds : UInt32): Duration`: Return a new `Duration` after subtracting the provided number of seconds from it.
- `subtractMiliseconds(ms : UInt32): Duration`: Return a new `Duration` after subtracing the provided number of miliseconds from it.
- `subtractMicroseconds(microsecs : UInt32): Duration`: Return a new `Duration` after subtracing the provided number of microseconds from it.

#### Examples

Example usage of `Duration` are

```cadence
let t1 = LocalDateTime.fromTimestamp(1415829132)
/// Assume that the current block timestamp is 1707045210
let t2 = LocalDateTime.now()
let d = Duration.betweenLocalDateTime(t1, t2)
d.getDays()         // 3370
d.getSeconds()      // 48078
d.getMicroseconds() // 0

let d2 = Duration.of(0, 365, 0, 0, 0, 0, 0) // 365 days
let d3 = Duration.of(1, 0, 0, 0, 0, 0, 0)   // 1 year
d2.getDays() // 365
d3.getDays() // 365

// Refer to the normalization rules. Only days can be negative.
let normalizedDuration = Duration.of(0, 0, 0, 0, 0, 0, -1) // -1 microseconds
normalizedDuration.getDays()         // -1
normalizedDuration.getSeconds()      // 86399
normalizedDuration.getMicroseconds() // 999999
```

### Drawbacks

None

### Alternatives Considered

None

### Performance Implications

None

### Dependencies

None

### Engineering Impact

It would require 3-4 weeks of engineering effort to implement, review & test the feature.

### Compatibility

This change has no impact on compatibility between systems (e.g. SDKs).

### User Impact

The proposed feature is a purely additive.
There is no impact on existing contracts and new transactions.

## Related Issues

None

## Questions and Discussion Topics

None

## Implementation
Will be done as part of https://github.com/onflow/cadence/issues/843.

