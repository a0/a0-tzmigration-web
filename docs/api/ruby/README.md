---
sidebar: auto
---

# Ruby gem

## Introduction

The [a0-tzmigration-ruby](https://rubygems.org/gems/a0-tzmigration-ruby) gem helps you automate the correction of inconsistencies when a new tzdb is released, given the following:
- your system stores all dates as unix timestamps, ie: seconds since the epoch ignoring leap seconds. Many current software works this way.
- your system/users choose a local representation that transform the above date to their local time. You should use an official tzdb timezone, like `America/Santiago`, or some mechanism to get that, UTC offsets like -04:00 are not enough.
- your system stores somewhere what version of the tzdb is currently being used, like `2018e`.

### Installation

Add this line to your application's Gemfile:

```ruby
gem 'a0-tzmigration-ruby'
```

And then execute:

```bash
$ bundle
```

Or install it yourself as:

```bash
$ gem install a0-tzmigration-ruby
```

### Basic Usage

Suppose your system is using tzdb version `2014j`, and a new version `2015a` is released. Once the sysadmin install the new version, users tell you there are inconsistencies.

Using this gem, you can query what changed for `America/Santiago` between versions `2014j` and `2015a`:

```ruby
require 'a0-tzmigration-ruby'

version_a = A0::TZMigration::TZVersion.new('America/Santiago', '2014j')
version_b = A0::TZMigration::TZVersion.new('America/Santiago', '2015a')

version_a.changes(version_b)
# =>
# [{:ini_str=>"2015-04-26 03:00:00 UTC", :fin_str=>"2015-09-06 04:00:00 UTC", :off_str=>"+01:00:00", :ini=>1430017200, :fin=>1441512000, :off=>3600},
#  {:ini_str=>"2016-04-24 03:00:00 UTC", :fin_str=>"2016-09-04 04:00:00 UTC", :off_str=>"+01:00:00", :ini=>1461466800, :fin=>1472961600, :off=>3600},
#  … (ommited) …
#  {:ini_str=>"2063-04-29 03:00:00 UTC", :fin_str=>"2063-09-02 04:00:00 UTC", :off_str=>"+01:00:00", :ini=>2945041200, :fin=>2955931200, :off=>3600},
#  {:ini_str=>"2064-04-27 03:00:00 UTC", :fin_str=>"∞", :off_str=>"+01:00:00", :ini=>2976490800, :fin=>Infinity, :off=>3600} ]
```

The resulting changes is an array of objects, where:
- `ini`, `fin` are unix timestamps or `-Float::INFINITY`, `Float::INFINITY`.
- `off` is seconds to add, or substract if negative.
- `ini_fin`, `fin_str` are the same ini, fin as UTC strings date, or `-∞`, `∞`.
- `off_str` is the same off as string.

The same changes in a table:

| ini_str                  |  fin_str                  |  off_str    |  ini         |  fin         |  off  |
|------------------------- | ------------------------- | ----------- | ------------ | ------------ | ------|
| 2015-04-26 03:00:00 UTC  |  2015-09-06 04:00:00 UTC  |  +01:00:00  |  1430017200  |  1441512000  |  3600 |
| 2016-04-24 03:00:00 UTC  |  2016-09-04 04:00:00 UTC  |  +01:00:00  |  1461466800  |  1472961600  |  3600 |
| … (ommited) … |
| 2063-04-29 03:00:00 UTC  |  2063-09-02 04:00:00 UTC  |  +01:00:00  |  2945041200  |  2955931200  |  3600 |
| 2064-04-27 03:00:00 UTC  |  ∞                        |  +01:00:00  |  2976490800  |  Infinity    |  3600 |

For each change, the rule to search affected dates in your system should be the following: \
if **ini** ≤ ***date*** < **fin** then **date** += **off** \
English: for dates between **ini** (included) and **fin** (excluded), add **off** to them.

Using our example, here are some changes that may be applied:
- For dates between 2015-04-26 03:00:00 UTC and 2015-09-06 04:00:00 UTC, add 1 hour.
- For dates between 2064-04-27 03:00:00 UTC and ∞, add 1 hour.

**You should always make sure you are comparing unix timestamps, or quering your system using UTC**. Never convert the above timestamps to localtime.

To convert a `Time` ruby object to unix timestamp, you can do the following:

```ruby
date = Time.new
timestamp = date.to_i
# => 1528057688
```

To convert a unix timestamp to a `Time` ruby object, do the following:

```ruby
timestamp = 1472961600

date = Time.at(timestamp)       # note this is localtime
# => 2016-09-04 01:00:00 -0300

date = Time.at(timestamp).utc   # prefer this, UTC time
# => 2016-09-04 04:00:00 UTC
```

### Other use cases

You can calculate changes between any *(timezone, version)*, not only consecutive transitions of a timezone. This is useful if the tzdb has not been updated regularly in your system, say you need to check the changes for *(America/Santiago, 2015a)* → *(America/Santiago, 2017a)*.

Maybe your users need to change their timezone. This happened in my country in 2017: a new timezone America/Punta_Arenas was created for the south region only. So, we needed to check the changes for *(America/Santiago, 2015a)* → *(America/Punta_Arenas, 2017a)*.

```ruby
require 'a0-tzmigration-ruby'

version_a = A0::TZMigration::TZVersion.new('America/Santiago', '2015a')
version_b = A0::TZMigration::TZVersion.new('America/Punta_Arenas', '2017a')

version_a.changes(version_b)
# => [{:ini_str=>"-∞", :fin_str=>"1890-01-01 04:43:40 UTC", :off_str=>"-00:00:54", :ini=>-Infinity, :fin=>-2524504580, :off=>-54},
#     {:ini_str=>"1910-01-01 04:42:46 UTC", :fin_str=>"1910-01-10 04:42:46 UTC", :off_str=>"+00:17:14", :ini=>-1893439034, :fin=>-1892661434, :off=>1034},
#     {:ini_str=>"1918-09-01 04:42:46 UTC", :fin_str=>"1918-09-10 04:42:46 UTC", :off_str=>"-00:42:46", :ini=>-1619983034, :fin=>-1619205434, :off=>-2566},
#     {:ini_str=>"1946-09-01 03:00:00 UTC", :fin_str=>"1947-04-01 04:00:00 UTC", :off_str=>"+01:00:00", :ini=>-736376400, :fin=>-718056000, :off=>3600},
#     {:ini_str=>"1947-05-22 04:00:00 UTC", :fin_str=>"1947-05-22 05:00:00 UTC", :off_str=>"+01:00:00", :ini=>-713649600, :fin=>-713646000, :off=>3600},
#     {:ini_str=>"1988-10-02 04:00:00 UTC", :fin_str=>"1988-10-09 04:00:00 UTC", :off_str=>"-01:00:00", :ini=>591768000, :fin=>592372800, :off=>-3600},
#     {:ini_str=>"1990-03-11 03:00:00 UTC", :fin_str=>"1990-03-18 03:00:00 UTC", :off_str=>"-01:00:00", :ini=>637124400, :fin=>637729200, :off=>-3600},
#     {:ini_str=>"2016-05-15 03:00:00 UTC", :fin_str=>"2016-08-14 04:00:00 UTC", :off_str=>"-01:00:00", :ini=>1463281200, :fin=>1471147200, :off=>-3600}]
```

If you want to create an UI, you can query the index of currently known timezones:

```ruby
# get current known timezones in the repository
A0::TZMigration::TZVersion.timezones
# => { "Africa/Abidjan" =>
#      { "versions" => [ "2013c", "2013d", "2013e", "2013f", "2013g", …
```

You can also query the index of currently known versions:

```ruby
# get current known versions in the repository
A0::TZMigration::TZVersion.versions
# => { "2013c" =>
#      { "released_at" => "2013-04-19 16:17:40 -0700",
#        "timezones" => [ "Africa/Abidjan", "Africa/Accra", "Africa/Addis_Ababa", "Africa/Algiers", "Africa/Asmara", …
```

### How it works

When you create a new TZVersion instance, it will download on demand the corresponding timezone file from our [tzdb version Repository](https://a0.github.io/a0-tzmigration-ruby/data/), which is documented here: [About the Repo](../../data/).

Your system needs to have access to the internet in order to download that file. The file is cached once per instance.

When you call `version_a.changes(version_b)`, it will compare the transitions of both versions and returns an array of changes.

The indexes are never cached in `TZVersion.timezones` or `TZVersion.versions`, but you may have a proxy.

## Configuration

### base_url

- default: `'https://a0.github.io/a0-tzmigration-ruby/data/'`

The URL from where files of the tzdb version repository will be downloaded. It must end with a `/`.  

```ruby
A0::TZMigration.configure do |config|
  config.base_url = 'https://127.0.0.1/my/repo/'
end

version = A0::TZMigration::TZVersion.new('America/Santiago', '2018e')
version.data # will fetch 'https://127.0.0.1/my/repo/timezones/America/Santiago.json'
```

## TZVersion instance methods

### initialize(name, version)

- parameters:
  - **name**: the name of the timezone, like `'America/Santiago'`
  - **version**: the version of the tzdb release, like `'2018e'`

Creates a new TZVersion, for the given name and version.

```ruby
version = A0::TZMigration::TZVersion.new('America/Santiago', '2018e')
```

### data

- returns: `Hash`
- cached: yes

Returns and caches the contents of the [Repository JSON file](../../../data/#the-timezones-json-structure), as read from the internet.

```ruby
version = A0::TZMigration::TZVersion.new('America/Santiago', '2018e')
version.data
# => {"name"=>"America/Santiago", "versions"=>{"2013c"=>{"tag"=>"v1.2013.3", "released_at"=>"2013-04-19 16:17:40 -0700", "transitions"=>[{"utc_offset"=>-16966, "utc_prev_offset"=>-16966, "utc_timestamp"=>-2524504634, "utc_time"=>"1890-01-01T04:42:46+00:00", "local_ini_str"=>"1890-01-01 00:00:00 LMT", "local_fin_str"=>"1890-01-01 00:00:00 SMT"}, …
```

### version_data

- returns: `Hash`
- cached: yes
- throws: `RuntimeError` if the version was not found in the data.

Returns and caches the corresponding value to the versions dictionary from the current data.

If this version is an alias to another timezone, that timezone is fetched and the version_data of that timezone is returned.

```ruby
version = A0::TZMigration::TZVersion.new('America/Santiago', '2018e')
version.version_data
# => {"tag"=>"v1.2018.5", "released_at"=>"2018-05-01 23:42:51 -0700", "transitions"=>[{"utc_offset"=>-16966, "utc_prev_offset"=>-16966, "utc_timestamp"=>-2524504634, "utc_time"=>"1890-01-01T04:42:46+00:00", "local_ini_str"=>"1890-01-01 00:00:00 LMT", "local_fin_str"=>"1890-01-01 00:00:00 SMT"}, {"utc_offset"=>-18000, "utc_prev_offset"=>-16966, "utc_timestamp"=>-1892661434, "utc_time"=>"1910-01-10T04:42:46+00:00", "local_ini_str"=>"1910-01-10 00:00:00 SMT", "local_fin_str"=>"1910-01-09 23:42:46 -05"}, …
```

### released_at

- returns: `String`
- cached: yes

Returns and caches the corresponding released_at from the current version_data.

```ruby
version = A0::TZMigration::TZVersion.new('America/Santiago', '2018e')
version.released_at
# => 2018-05-01 23:42:51 -0700
```

### transitions

- returns: `Array`
- cached: yes

Returns and caches the corresponding [transitions](../../../data/#transitions) from the current version_data.

```ruby
version = A0::TZMigration::TZVersion.new('America/Santiago', '2018e')
version.transitions
# => 
# [{"utc_offset"=>-16966, "utc_prev_offset"=>-16966, "utc_timestamp"=>-2524504634, "utc_time"=>"1890-01-01T04:42:46+00:00", "local_ini_str"=>"1890-01-01 00:00:00 LMT", "local_fin_str"=>"1890-01-01 00:00:00 SMT"}
#  {"utc_offset"=>-18000, "utc_prev_offset"=>-16966, "utc_timestamp"=>-1892661434, "utc_time"=>"1910-01-10T04:42:46+00:00", "local_ini_str"=>"1910-01-10 00:00:00 SMT", "local_fin_str"=>"1910-01-09 23:42:46 -05"}
#  …
#  {"utc_offset"=>-10800, "utc_prev_offset"=>-14400, "utc_timestamp"=>3080520000, "utc_time"=>"2067-08-14T04:00:00+00:00", "local_ini_str"=>"2067-08-14 00:00:00 -04", "local_fin_str"=>"2067-08-14 01:00:00 -03"}
#  {"utc_offset"=>-14400, "utc_prev_offset"=>-10800, "utc_timestamp"=>3104103600, "utc_time"=>"2068-05-13T03:00:00+00:00", "local_ini_str"=>"2068-05-13 00:00:00 -03", "local_fin_str"=>"2068-05-12 23:00:00 -04"} ]
```

The transitions are sorted.

### transition_ranges

- returns: `Array`
- cached: yes

Returns and caches a generated transition ranges from the current version_data.

A transition range represents a continuos time where this timezone has the same offset from UTC. Is an array of dictionaries with the following keys:

- `ini` is the starting time (inclusive) as a unix timestamp or `-Float::INFINITY`.
- `fin` is the ending time (exclusive) as a unix timestamps or `Float::INFINITY`.
- `off` is the number of seconds offset from UTC.
- `ini_fin`, `fin_str` are the same ini, fin as UTC strings date, or `-∞`, `∞`.
- `off_str` is the same off as a string.

```ruby
version = A0::TZMigration::TZVersion.new('America/Santiago', '2018e')
version.transition_ranges
# => 
# [{:ini_str=>"-∞", :fin_str=>"1890-01-01 04:42:46 UTC", :off_str=>"-04:42:46", :ini=>-Infinity, :fin=>-2524504634, :off=>-16966},
#  {:ini_str=>"1890-01-01 04:42:46 UTC", :fin_str=>"1910-01-10 04:42:46 UTC", :off_str=>"-04:42:46", :ini=>-2524504634, :fin=>-1892661434, :off=>-16966},
#  …
#  {:ini_str=>"2067-08-14 04:00:00 UTC", :fin_str=>"2068-05-13 03:00:00 UTC", :off_str=>"-03:00:00", :ini=>3080520000, :fin=>3104103600, :off=>-10800},
#  {:ini_str=>"2068-05-13 03:00:00 UTC", :fin_str=>"∞", :off_str=>"-04:00:00", :ini=>3104103600, :fin=>Infinity, :off=>-14400} ]
```

The transition ranges are sorted.

### timestamps

- returns: `Array`
- cached: yes

Returns and caches the list of timestamps at which transitions start/end, adding `-Float::INFINITY` at the beginning and `Float::INFINITY` at the end.

```ruby
version = A0::TZMigration::TZVersion.new('America/Santiago', '2018e')
version.timestamps
# => 
# [-Infinity,
#  -2524504634,
#  …
#  3104103600,
#  Infinity]
```

The timestamps are sorted.

### changes

- parameters:
  - **other**: another TZVersion instance.
- returns: `Array`
- cached: no

Calculates the changes between this version an the other version, based on the transition_ranges of each one.

```ruby
require 'a0-tzmigration-ruby'

version_a = A0::TZMigration::TZVersion.new('America/Santiago', '2018e')
version_b = A0::TZMigration::TZVersion.new('America/Punta_Arenas', '2018e')

version_a.changes(version_b)
# =>
# [{:ini_str=>"-∞", :fin_str=>"1890-01-01 04:43:40 UTC", :off_str=>"-00:00:54", :ini=>-Infinity, :fin=>-2524504580, :off=>-54},
#  {:ini_str=>"1946-07-15 04:00:00 UTC", :fin_str=>"1946-09-01 03:00:00 UTC", :off_str=>"-01:00:00", :ini=>-740520000, :fin=>-736376400, :off=>-3600},
#  … (ommited) …
#  {:ini_str=>"2067-05-15 03:00:00 UTC", :fin_str=>"2067-08-14 04:00:00 UTC", :off_str=>"+01:00:00", :ini=>3072654000, :fin=>3080520000, :off=>3600},
#  {:ini_str=>"2068-05-13 03:00:00 UTC", :fin_str=>"∞", :off_str=>"+01:00:00", :ini=>3104103600, :fin=>Infinity, :off=>3600}]
```

The changes are sorted.

## TZVersion static methods

### versions

- returns: `Hash`
- cached: no

Returns the contents of the [versions index](../../../data/#the-versions-index-json-structure), as read from the internet.

```ruby
A0::TZMigration::TZVersion.versions()
# =>
# {"2013c"=>
#   {"released_at"=>"2013-04-19 16:17:40 -0700",
#    "timezones"=> ["Africa/Abidjan", "Africa/Accra", "Africa/Addis_Ababa", "Africa/Algiers", "Africa/Asmara", …
```

### timezones

- returns: `Hash`
- cached: no

Returns the contents of the [timezones index](../../../data/#the-timezones-index-json-structure), as read from the internet.

```ruby
A0::TZMigration::TZVersion.timezones()
# =>
# {"Africa/Abidjan"=>
#   {"versions"=>
#     ["2013c", "2013d", "2013e", "2013f", "2013g", "2013h", …
```
