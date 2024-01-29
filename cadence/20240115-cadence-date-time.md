---
status: draft
flip: 242
authors: darkdrag00n (darkdrag00n@proton.me)
sponsor: Bastian Müller (bastian@dapperlabs.com)
updated: 2024-01-15
---

# FLIP 242: Date & Time

## Objective

This FLIP proposes adding Date & Time types and manipulation utilities to Cadence. It proposes two new types, `LocalDateTime` and `Duration` along with functions to create and manipulate them.

## Motivation

Manipulating Date & Time is a common aspect of user programs. Presently, Cadence users have to implement these calculations using raw `UFix64` timestamps.

Doing them manually is both hard and error-prone. As a result, standard library support similar to other modern languages would enhance the usability of the language.

## User Benefit

Manipulating date and time using raw timestamps is difficult and error-prone. Providing rich suite of standard library types and functions would increase the usability and user-experience.

## Design Proposal

New types `LocalDateTime` & `Duration` will be added to Cadence.

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

- `LocalDateTime.of(year: UInt64, month: UInt64, day: UInt64, hour: UInt64, minute: UInt64, second: UInt64)`: Creates a `LocalDateTime` with value based on the passed arguments assuming that there are 3600*24 seconds in every day.

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
// TODO

### Duration
A `Duration` will represent the difference between two dates or times. It will defined as follows:

```cadence
pub struct Duration {
    let timestamp_delta : Fix64  // Can be negative.
}
```

#### Constructor Functions
Constructor functions will be defined to instantiate variables of type `Duration`. They are defined below.

- `Duration.betweenLocalDateTime(t1 : LocalDateTime, t2 : LocalDateTime)`: Creates a `Duration` which represents the time duration between arguments `t1` and `t2`.

- `Duration.of(year: UInt64, month: UInt64, day: UInt64, hour: UInt64, minute: UInt64, second: UInt64)`: Creates a `Duration` with value based on the passed arguments assuming that there are 3600*24 seconds in every day.

#### Public Members
The `Duration` will provide the following public members:

- `getYear(): UInt32`: Get the year of the `Duration`.
- `getMonth(): Month`: Get the month of the `Duration`.
- `getDays(): UInt32`: Get the days of the `Duration`.
- `getHour(): UInt32`: Get the hour of the `Duration`.
- `getMinute(): UInt32`: Get the minute of the `Duration`.
- `getSecond(): UInt32`: Get the second of the `Duration`.
- `addYears(years : UInt32): Duration`: Return a new `Duration` after adding the provided number of years to it.
- `addMonths(months : UInt32): Duration`: Return a new `Duration` after adding the provided number of months to it.
- `addDays(days : UInt32): Duration`: Return a new `Duration` after adding the provided number of days to it.
- `addHours(hours : UInt32): Duration`: Return a new `Duration` after adding the provided number of hours to it.
- `addMinutes(minutes : UInt32): Duration`: Return a new `Duration` after adding the provided number of minutes to it.
- `addSeconds(seconds : UInt32): Duration`: Return a new `Duration` after adding the provided number of seconds to it.
- `subtractYears(years : UInt32): Duration`: Return a new `Duration` after subtracting the provided number of years from it.
- `subtractMonths(months : UInt32): Duration`: Return a new `Duration` after subtracting the provided number of months from it.
- `subtractDays(days : UInt32): Duration`: Return a new `Duration` after subtracting the provided number of days from it.
- `subtractHours(hours : UInt32): Duration`: Return a new `Duration` after subtracting the provided number of hours from it.
- `subtractMinutes(minutes : UInt32): Duration`: Return a new `Duration` after subtracting the provided number of minutes from it.
- `subtractSeconds(seconds : UInt32): Duration`: Return a new `Duration` after subtracting the provided number of seconds from it.

#### Examples

// TODO

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

