# Hive Funnel Analysis UDFs

**Please note**: This project is no longer maintained and contains known security vulnerabilities. Use at your own risk!

[![Build Status](https://travis-ci.org/yahoo/hive-funnel-udf.svg?branch=master)](https://travis-ci.org/yahoo/hive-funnel-udf)
[![Coverage Status](https://coveralls.io/repos/github/yahoo/hive-funnel-udf/badge.svg?branch=master)](https://coveralls.io/github/yahoo/hive-funnel-udf?branch=master)
[![Apache License 2.0](https://img.shields.io/badge/license-Apache%202.0-blue.svg?style=flat)](LICENSE)

[Funnel analysis](https://en.wikipedia.org/wiki/Funnel_analysis) is a method for
tracking user conversion rates across actions. This enables detection of actions
causing high user fallout.

These Hive UDFs enables funnel analysis to be performed simply and easily on any
Hive table.

## Table of Contents

  * [Requirements](#requirements)
  * [How to build](#how-to-build)
    * [Build JAR](#build-jar)
    * [Register JAR with Hive](#register-jar-with-hive)
  * [How to use](#how-to-use)
    * [`funnel`](#funnel)
    * [`funnel_merge`](#funnel_merge)
    * [`funnel_conversion`](#funnel_conversion)
    * [`funnel_fallout`](#funnel_fallout)
  * [Examples](#examples)
    * [Simple funnel](#simple-funnel)
    * [Simple funnel with conversion](#simple-funnel-with-conversion)
    * [Funnel with multiple groups](#funnel-with-multiple-groups)
    * [Multiple parallel funnels](#multiple-parallel-funnels)
  * [Contributors](#contributors)
  * [License](#license)

## Requirements

[Maven](https://maven.apache.org/index.html) is required to build the funnel
UDFs.

## How to build

There is a provided `Makefile` with all the build targets.

### Build JAR

```bash
make jar
```

This creates a `funnel.jar` in the `target/` directory.

### Register JAR with Hive

To use the funnel UDFs, you need to register it with Hive.

With temporary functions:

```sql
ADD JAR funnel.jar;
CREATE TEMPORARY FUNCTION funnel            AS 'com.yahoo.hive.udf.funnel.Funnel';
CREATE TEMPORARY FUNCTION funnel_merge      AS 'com.yahoo.hive.udf.funnel.Merge';
CREATE TEMPORARY FUNCTION funnel_conversion AS 'com.yahoo.hive.udf.funnel.Conversion';
CREATE TEMPORARY FUNCTION funnel_fallout    AS 'com.yahoo.hive.udf.funnel.Fallout';
```

With permenant functions you need to put the JAR on HDFS, and it will be registered with a database (you have to replace `DATABASE` and `PATH_TO_JAR` with your values):

```sql
CREATE FUNCTION DATABASE.funnel            AS 'com.yahoo.hive.udf.funnel.Funnel'  USING JAR 'hdfs:///PATH_TO_JAR/funnel.jar';
CREATE FUNCTION DATABASE.funnel_merge      AS 'com.yahoo.hive.udf.funnel.Merge'   USING JAR 'hdfs:///PATH_TO_JAR/funnel.jar';
CREATE FUNCTION DATABASE.funnel_conversion AS 'com.yahoo.hive.udf.funnel.Conversion' USING JAR 'hdfs:///PATH_TO_JAR/funnel.jar';
CREATE FUNCTION DATABASE.funnel_fallout    AS 'com.yahoo.hive.udf.funnel.Fallout' USING JAR 'hdfs:///PATH_TO_JAR/funnel.jar';
```

## How to use

There are fout funnel UDFs provided: [`funnel`](#funnel),
[`funnel_merge`](#funnel_merge), [`funnel_conversion`](#funnel_conversion),
[`funnel_fallout`](#funnel_fallout).

The [`funnel`](#funnel) UDF outputs an array of longs showing conversion rates
across the provided funnel steps.

The [`funnel_merge`](#funnel_merge) UDF merges multiple arrays of longs by
adding them together.

The [`funnel_conversion`](#funnel_conversion) UDF takes a raw count funnel result and
converts it to the conversion rate.

The [`funnel_fallout`](#funnel_fallout) UDF takes a raw count funnel result and
converts it to the fallout rate.

There is no need to sort the data on timestamp, the UDF will take care of it. If
there is a collision in the timestamps, it then sorts on the action column.

### `funnel`
`funnel(action_column, timestamp_column, array(funnel_1_a, funnel_1_b), array(funnel_2), ...)`
  - Builds a funnel report applied to the `action_column`, sorted by the
    `timestamp_column`.
  - The funnel steps are arrays of the same type as the `action` column. This allows
    for multiple matches to move to the next funnel.
    - For example, funnel_1 could be `array('register_button',
      'facebook_invite_register')`. The funnel will match the first occurence
      of either of these actions and proceed to the next funnel.
    - Or, funnel_1 could just be `array('register_button')`.
  - You can have an arbitrary number of funnels.
  - The `timestamp_column` can be of any comparable type (Strings, Integers,
    Dates, etc).

### `funnel_merge`
`funnel_merge(funnel_column)`
  - Merges funnels. Use with funnel UDF.

### `funnel_conversion`
`funnel_conversion(funnel_column)`
  - Converts the result of a [`funnel_merge`](#funnel_merge) to a conversion
    rate.  Use with funnel and funnel_merge UDF.
  - For example, a result from [`funnel_merge`](#funnel_merge) could look like
    `[245, 110, 54, 13]`. This is result is in raw counts. If we pass this
    through [`funnel_conversion`](#funnel_conversion) then it would look like
    `[1.0, 0.44, 0.49, 0.24]`.

### `funnel_fallout`
`funnel_fallout(funnel_column)`
  - Converts the result of a [`funnel_merge`](#funnel_merge) to a fallout rate.
    Use with funnel and funnel_merge UDF.
  - For example, a result from [`funnel_merge`](#funnel_merge) could look like
    `[245, 110, 54, 13]`. This is result is in raw counts. If we pass this
    through [`funnel_fallout`](#funnel_fallout) then it would look like `[0.0,
    0.55, 0.50, 0.75]`.

## Examples

Assume a table `user_data`:

| action              | timestamp | user_id | gender |
|---------------------|-----------|---------|--------|
| signup_page         | 100       | 1       | f      |
| confirm_button      | 200       | 1       | f      |
| submit_button       | 300       | 1       | f      |
| signup_page         | 200       | 2       | m      |
| submit_button       | 400       | 2       | m      |
| signup_page         | 100       | 3       | f      |
| confirm_button      | 200       | 3       | f      |
| decline             | 200       | 3       | f      |
| ...                 | ...       | ...     | ...    |

### Simple funnel

```sql
SELECT funnel_merge(funnel)
FROM (SELECT funnel(action, timestamp, array('signup_page', 'email_signup'),
                                       array('confirm_button'),
                                       array('submit_button')) AS funnel
      FROM user_data
      GROUP BY user_id) t1;
```

Result: `[3, 2, 1]`

### Simple funnel with conversion rate

```sql
SELECT funnel_conversion(funnel_merge(funnel))
FROM (SELECT funnel(action, timestamp, array('signup_page'),
                                       array('confirm_button'),
                                       array('submit_button')) AS funnel
      FROM user_data
      GROUP BY user_id) t1;
```

Result: `[1.0, 0.66, 0.5]`

### Funnel with multiple groups

```sql
SELECT gender, funnel_merge(funnel)
FROM (SELECT gender,
             funnel(action, timestamp, array('signup_page'),
                                       array('confirm_button'),
                                       array('submit_button')) AS funnel
      FROM table
      GROUP BY user_id, gender) t1
GROUP BY gender;
```

Result: `m: [1, 0, 0], f: [2, 2, 1]`

### Multiple parallel funnels

```sql
SELECT funnel_merge(funnel1), funnel_merge(funnel2)
FROM (SELECT funnel(action, timestamp, array('signup_page'),
                                       array('confirm_button'),
                                       array('submit_button')) AS funnel1
             funnel(action, timestamp, array('signup_page'),
                                       array('decline')) AS funnel2
      FROM table
      GROUP BY user_id) t1;
```

Result: `[3, 2, 1] [3, 1]`

## Contributors

Josh Walters, [josh@joshwalters.com](mailto:josh@joshwalters.com)

## License

[Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0)
