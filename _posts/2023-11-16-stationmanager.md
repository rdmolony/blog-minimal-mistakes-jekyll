---
title: "Struggling to sync sensors & databases"
layout: post
---

Over 2022/23 I worked at `Mainstream Renewable Power` on an internal web application called `StationManager` used by the `Energy Analysis Group` (meteorological station managers & energy analysts).

Its job -

- Data Access[^RAB]

  [^RAB]: Sensor readings are tracked by loggers.  Loggers save readings to text files.  `StationManager` fetches these files from the remote sensors, amalgamates readings into a file format suitable for other tools & makes these files accessible via a friendly web application user interface.

- Sensor Health Monitoring[^CHI]

  [^CHI]: The team need to know if the sensors are syncing or if the sensors faulty, so meteorological station managers can go on-site & fix them if needs be.  As well as flagging data quality automatically, it provides a user interface for manually flagging erroneous readings.

- Exploratory `Jupyter Notebook` driven data analysis[^FOO]
  
  [^FOO]: In cases where a particular analysis is not yet well served by 3rd party tooling,  the team uses a shared `Jupyter Notebook` server to explore & visualise data in `Python`

Its core value lies in fetching files from remote sensors, transforming them into useful file outputs[^CIE] -

[^CIE]: in most cases a `Windographer` file, a 3rd party tool used to analyze & visualize wind resource data


![sensors-to-useful-files.svg](/assets/images/2023-11-16-stationmanager/sensors-to-useful-files.svg)


So how does it work?  And more importantly, how can we ensure that it works?

> In each section **v1** indicates before my time & **v2** after

- [Fetching files from remote sensors](#fetching-files-from-remote-sensors)
- [Importing sensor files to a database](#importing-sensor-files-to-a-database)
- [Cleaning sensor readings](#cleaning-sensor-readings)
- [Accessing data](#accessing-data)
- [Handling issues](#handling-issues)
  - [Tests](#tests)
  - [Alerting](#alerting)



> In 2014/15 a brave individual called Paul Hughes managed to pull the bulk of a system together.  It then passed through the hands of three more people (Sean Hayes, Andrew McGregor & Tomasz Jama-Lipa) before reaching me,  with each person adding their own twist to keep it alive & make it useful.

> I am grateful for the guidance, support & faith provided by my manager Romain Molins these last two years, & to work with Piotr Szczesniak this last year who will proceed me.


---
<br>


# Fetching files from remote sensors

**v1** -

For `Campbell Scientific` loggers,  it's quite straightforward[^RAT].  Just install `LoggerNet` on a `Microsoft Windows` server,  configure the loggers & tada the files are fetched & saved to the server.

[^RAT]: At least from a software engineer's perspective, installing loggers & configuring `LoggerNet` is not something I have experience with

`LoggerNet` only works with `Campbell Scientific` loggers.

What about the other loggers?

Remote sensors are normally in deserts so connecting to them is not cheap or easy.

So a 3rd party, [`SmartGrid Technologies`](https://www.igrid.co.za/), was hired to sync files from loggers to a remote filesystem,  which then need to be synced to the `LoggerNet` server.

So a `Python` job was created to automatically sync the two filesystems via `SFTP` (i.e. Secure File Transfer Protocol).

These files are normally compressed & encrypted.  To process them,  they must first be standardised to text files.

So a `Python` job was created to automatically unzip & decrypt files.


![sensors-to-loggernet-server.svg](/assets/images/2023-11-16-stationmanager/sensors-to-loggernet-server.svg)


The `Python` jobs were executed via `Task Scheduler` on the `LoggerNet` server as `Python` scripts in which credentials & file paths[^XOO] were hard-coded.

[^XOO]: To sync files from one file system to another one needs to know where it is stored on each filesystem!

**v2** -

My changes were small -

- I pulled credentials & file paths into a `TOML` file to keep them outside of source-control & make them easier to edit
- I adapted the scripts into notebooks to make introspection of code & outputs easier.  Each notebook does one thing[^XAN] & each one is run using [`papermill`](https://github.com/nteract/papermill) via `Task Scheduler`

[^XAN]: One for unzipping, one for decrypting, one for file syncing ...

There wasn't much more to be done here since the loggers are configured on-site to use either `LoggerNet` or the `SmartGrid` connection.


---
<br>


# Importing sensor files to a database

**v1** -

How to amalgamate all sensor files to one place?

One could -

1. Use a library that can read from multiple files each time the files are processed
2. Create a file of files which copies all files to one place
3. Import the readings from each file to a database table

`Campbell Scientific` provide an application called `LNDB` which automatically exports sensor readings to database tables,  so it makes sense that the original creator opted for option 3 using the `Microsoft SQL Server` database.

Each logger gets its own database table which look like ...

| timestamp | sensor_name_1 | ... | sensor_name_n |
| --- | --- | --- | --- |
| value | value | ... | value |

What about the other loggers?

No off-the-shelf tool existed.  So `Python` to the rescue.

Reading text files & importing them to a database table is surprisingly hard -

- If it includes metadata, it needs to be skipped
- If it isn't encoded in `utf-8`, it needs to be specified as this is hard to infer
- If it includes a column of timestamp strings, these need to be standardised
- If it contains a column that does not exist in the corresponding database table, the database table needs to be edited to include it

Each type of logger has its own conventions so each type necessitated its own `Python` file importer.

How to track which file has been imported & which has not?

Another database, a sensor metadata database, was linked to the `Python` job to track imports.

As a nice bonus,  users could then view file import status using the `Django` web-based user interface (discussed later) since it is connected to this database.


![loggernet-server-to-mssql-database.svg](/assets/images/2023-11-16-stationmanager/loggernet-server-to-mssql-database.svg)


**v2** -

Now for some heftier changes.

What if instead of storing the readings for each logger in its own table the readings are standardised before storage?

So instead of storing each source in its own table like ...

| timestamp | sensor_name_1 | ... | sensor_name_m |
| --- | --- | --- | --- |
| value | value | ... | value |

...

| timestamp | sensor_name_n | ... | sensor_name_m |
| --- | --- | --- | --- |
| value | value | ... | value |

... create a single table of sources like ...

| timestamp | sensor_id | value |
| --- | --- | --- |
| value | value | value |

... so all readings for all sources can be queried from one place.

Won't this be really slow to store & query since the table will contain **a lot**[^YAT] of readings?

[^YAT]: Typically each logger records average, standard deviation, minimum & maximum values every 10 minutes for each sensor.  If a logger linked to 20 sensors records for 6 years, then it will be produce `20 sensors * 4 reading types * 6 readings/hour * 24 hours * 365 days * 6 years = 25,228,800 readings`.

Yes.  In a traditional relational database like  `Microsoft SQL Server`, `Postgres` or `MySQL` it will be,  since these databases are designed to store & query new entries row by row.

What about timeseries databases?

`TimescaleDB` is a `Postgres` extension that enables working with timeseries readings from within `Postgres`[^NAT].  It provides a special table called a `Hypertable` which speeds up insert & query performance for timeseries tables[^CAT]. 

[^NAT]: It's a very popular database, see [StackOverFlow's 2023 Developer Survey](https://survey.stackoverflow.co/2023/#section-most-popular-technologies-databases)
[^CAT]: As of 2023-11-20 `TimescaleDB` by default chunks readings in compressed hypertables into intervals of one week so `Postgres` doesn't need to read all rows for queries that only care about a particular period of time, see [Hypertables](https://docs.timescale.com/use-timescale/latest/hypertables/)

I went all in on `TimescaleDB`.

Since `Django` works well with `Postgres` I could also use `TimescaleDB` to store all sensor metadata,  so two databases became one -


![loggernet-server-to-timescale-database.svg](/assets/images/2023-11-16-stationmanager/loggernet-server-to-timescale-database.svg)


This switch didn't come for free since I now need a more intelligent importer for all sources since data needs to be transformed into the right format before storing it,  and this process depends on the source!

Having said that,  I figured the additional complexity on import was worth the cost given that processing logic on this table is now standardised for all sources.


---
<br>


# Cleaning sensor readings

**v1** -

By default all of the loggers store sensor readings in generic calibrations, so each reading needs to be re-calibrated using its specific calibration before it can be used in analysis.

> An anemometer measures wind speeds by counting the number of rotations per second or frequency of its spinning cups.  To convert frequency to wind speed we need to know an anemometer's calibration[^XAR].

[^XAR]:  Calibrations are measured in a wind tunnel.  If one blows wind over an anemometer for various wind speeds & measures the number of rotations for each then one can infer it.  Specifically, calibrations are the values for slope, `m`, and offset, `c`, in `f(x) = mx + c` where `x` refers to frequency &  `f(x)` to wind speed


![anemometer.svg](/assets/images/icons/anemometer.svg){: width="100" }


Sometimes there are issues with a sensor, so its readings are not valid & need to be removed.

How to re-calibrate & clean millions of readings?

`pandas`

> `pandas` is a `Python` library for reading, manipulating & writing data.

This one's hairy.


![mssql-database-to-useful-files.svg](/assets/images/2023-11-16-stationmanager/mssql-database-to-useful-files.svg)


`Task Scheduler` periodically runs a job in which all timeseries operational stations are updated -

1. `pandas` asks the timeseries database for readings via `SQLAlchemy` (powered by `pyodbc` & `ODBC Driver for SQL Server 17`)
2. `pandas` asks the sensor metadata database for sensor metadata & user-specified erroneous reading timestamps via `SQLAlchemy` (powered by `mysqlclient`)
3. `pandas` caches readings to a `pickle` file to skip trips to the timeseries database if specified
4. `pandas` re-calibrates sensor readings & filters out erroneous ones using rules & user-specified "flags"

It's hard to test this glue since it depends on two databases & cache files.  It's also tricky managing the database connections[^TAD].

[^TAD]: The connections to the timeseries database are stored in the sensor metadata database ...

Re-calibrating & flagging the readings in a format like ...

| timestamp | sensor_name_1 | ... | sensor_name_n |
| --- | --- | --- | --- |
| value | value | ... | value |

... is surprisingly difficult since each column name is also data - it implies what type of sensor the column of data represents.

So if one has to thread carefully to process it -

```ruby
# sensor looks like timestamp_1=value, ...
for sensor in sensor_name_1...sensor_name_n:

  # If a sensor has been replaced
  # then `pandas` must apply different calibrations 
  # to different timestamp ranges
  for timestamps, recalibrate in calibrations[sensor]:
    sensor[timestamps] = recalibrate sensor[timestamps]

  # Flag rules differ depending on the type of data -
  # a very large or very small wind speed reading will differ
  # from a very large battery voltage
  for flag_rule in flag_rules[sensor]:
    for filter_by_flag_rule in flag_rule:
      sensor = filter_by_flag_rule sensor

  # User "flags" specify that say a wind speed sensor was dodgy
  # for a few days as it was broken so only apply to some columns
  # for some timestamps
  for timestamps, filter_by_user_flag in user_flags[sensor]:
    sensor[timestamps] = filter_by_user_flag sensor[timestamps]
```

**v2** -


![timescale-database-to-useful-files.svg](/assets/images/2023-11-16-stationmanager/timescale-database-to-useful-files.svg)


It's shockingly easier to work with the data once it's reformatted to a "tidier"[^ROR] format ...

[^ROR]: See [Hadley Wickham's Tidy Data](https://vita.had.co.nz/papers/tidy-data.pdf) for a more in-depth discussion.  Note that the "tidier" representation is still not "tidy" since the values column doesn't necessarily represent a single unit like wind speed in m/s or wind direction in degrees but rather contains many units!

| timestamp | sensor_id | value |
| --- | --- | --- |
| value | value | value |

Now that each row represents a unique reading,  the data is "mergeable"!

So I can link each reading with its corresponding metadata ...

| timestamp | sensor_id | value | calibration | flag_rule | user_flag |
| --- | --- | --- | --- | --- | --- |
| value | value | value | value | value | value |

... and processing becomes much simpler since I can now apply each rule directly rather than over a range of timestamps & a range of columns since metadata is already linked to its corresponding timestamp -

```
reading = reading -> calibrate -> filter_by_flag_rule -> filter_by_user_flag
```

Will this scale to any size dataset?

As of 2023, `pandas` loads all data into memory by default.  If one wants to process more than one set of readings at the same time & each set of readings number in the millions this can push a server to its limits.

This particularly becomes an issue on merging readings with metadata since the data grows & grows in size[^XKC].

[^XKC]: This might not remain an issue forever, as of this writing `polars` is capable of "streaming" data from files, so only reading what it needs in chunks.  `pandas` does have `chunksize` for reading csvs, however, I've found it clunky to work with when compared to `polars` or `SQL`

I had to maintain a `Django` web application & this requires a database, so why not leverage the database "engine" for processing data?

Databases are smart.  They don't load all data into memory by default & so can handle these merges without using up all server resources, but they do have other complexities.

As discussed,  I decided to store readings in the database via `TimescaleDB`.

With the readings living alongside the sensor metadata in `TimescaleDB` merging & processing is straightforward -

<details>
<summary>👈 Re-calibrate via SQL</summary>
<br>
{% highlight sql %}
      select  reading.timestamp,
              reading.file_id,
              reading.sensor_id,
              (
                case
                  when meta.data_type = 'Wind Direction'
                       and meta.reading_type = 'avg'
                  then float_modulus(
                         reading.generic_reading
                         + meta.deadband
                         + case
                             when meta.oriented_to_true_north = false
                             then meta.magnetic_declination
                             else 0
                           end,
                         360
                       )
                  else reading.generic_reading
                end
                - meta.logger_offset
              )
              * (meta.certificate_slope / meta.logger_slope)
              + meta.certificate_offset
              as calibrated_reading

        from  sensor_reading_generic as reading
  inner join  sensor_metadata  as meta
          on  meta.sensor_name_id = reading.sensor_id
              and reading.timestamp >= meta.sensor_commission_date
              and reading.timestamp < meta.sensor_decommission_date
  inner join  logger_file as file
          on  file.id = reading.file_id
              and file.station_id = reading.station_id
              and file.is_ignored = false

        where reading.station_id = <STATION_ID>
              and meta.station_id = <STATION_ID>
              and reading.timestamp >= '<START>'::timestamp
              and reading.timestamp < '<STOP>'::timestamp
              and flag.type is distinct from 'Fault'
{% endhighlight %}
</details>
<br>

What's not straightforward, however, is making file exports fast -

- Big queries can be slow.  They read from disk as well as memory - unlike `pandas` - which is good for server resources but worse for performance.
- Reformatting query results in `SQL` is hard.  Queries are read in batches & reformatted in `Python`[^EEK]

[^EEK]: I found it simpler to reformat readings from one reading per row to one column per sensor for the output file in `Python`.  `Postgres` has `crosstab` for going from wide to long, but I found it hard to work work with since my queries were dynamic crosstabulations which it doesn't handle easily.  I can also use `max(case when sensor_id=<SENSOR_ID> then reading else null) from reading group by timestamp` to achieve the same result as `crosstab`.  Both methods let me use `COPY TO` to save to a file directly in `SQL`, which can be really fast, but I found exporting to formats other than plain text files like `zip` hard & require more work back in `Python`.

How to speed up queries?

If you don't have the right indexes for your query `TimescaleDB` won't go fast.

[Finding the right indexes](
https://www.timescale.com/blog/use-composite-indexes-to-speed-up-time-series-queries-sql-8ca2df6b3aaa/
) was't too bad once I realised I could wrap my slow queries in `EXPLAIN ANALYSE` & experiment on different indexes to see the impact of each.

How to export multiple sources in parallel?

A task queue.

A task queue works like a restaurant.  The waiters add an order to the queue & the chefs pull orders from the queue when they have time to process it.

Previously each set of readings was processed one by one & only two types of "useful" output file was produced per source.  Now 10 timeseries output files are produced per source,  & producing each one is slower.  With some parallel processing this becomes more manageable.

Managing database & task worker resource usage is hard.

Estimating the appropriate number of workers for the task queue is somewhat of a fine art.  I experimented with various numbers of "threads" & "processes" & queued the most CPU & RAM intensive tasks while watching resource usage to come up guess numbers.

How much CPU or RAM is too much?  Should minimising CPU or RAM be prioritised?

How to keep the number of `Postgres` connections under control?

Each worker runs a big database query, which may or may not leverage `Postgres` parallel workers such that one connection can become multiple.  I ran into trouble when my task queue workers exhausted the `Postgres` connection pool which caused the connected web application to crash.

I worked out that I could limit the number of connections by routing my `Postgres` connection through a connection pool via `PgBouncer`, which forces reusing connections rather than spinning up new ones.

`Postgres` may still spin up parallel workers if the query planner decides this is necessary.  So I had to experiment with `max_parallel_workers_per_gather` & `max_parallel_workers` in `postgresql.conf` to prevent these types of crashes.


---
<br>


# Accessing data

Now for the "visible" part, the web application -

![useful-files-to-user.svg](/assets/images/2023-11-16-stationmanager/useful-files-to-user.svg)

If the user wants to access or update data for a particular source,  they can search for the source in the web application & from there they can -

- Change the calibrations for a particular source's sensor
- Flag erroneous readings graphically
- Manually refresh a "useful" export file[^KUD]

[^KUD]: Their request is added to the task queue which is then processed by a worker when it has time to do so.  They might wish to do so if the calibrations of a station have changed or if a new "flags" marking erroneous readings have been added more recently than the most recent source cache.


---
<br>

# Handling issues

Even after a rearchitecture the system was still more complex than I would like.

So good tools for essential for managing this complexity -

## Tests

[Inspired by Harry Percival](https://www.obeythetestinggoat.com/) I setup a `pytest` test suite which grew to hundreds of tests.  Code tests check that code actually does what it was designed to do.

Getting started introducing tests to an untested code base is hard, especially when learning the tooling for the first time.  It's an unreasonable goal (for a human anyway) to aim for covering every class & function with a test,  so I opted first for high-level browser tests & then for lower-level unit tests.

My `Selenium` based browser tests were pretty dumb but effective.  They spin up a browser & click through the website & fail if anything breaks.  The catch is that the test won't normally give you a good reason **why something breaks**, just that it does indeed break.  I created one test per common use case. 

I backed up these browser tests with some fairly dumb unit tests which check a particularly important function or class doesn't break.  These tests don't require a browser & so are much faster to run.  Specifically, they check that given a valid input does the code give me a valid output.

As things break,  one can "catch" the breakage in a unit test, fix the code & then prove it works by passing the test.  As more & more things are fixed the test coverage grows.

**Tests are not everything**.  Breakages can still occur between systems, that won't be caught by tests.  For example,  if the database is setup differently to what the code expects then this will cause issues & won't be necessarily caught.  This is particularly true if rolling out changes isn't automated.

However,  tests do still provide an ease of mind.  I setup the tests to run on every code change via `GitHub Actions` & over time I got more comfortable that code changes were less likely to break things.

## Alerting

Good alerting is particularly important for a data pipeline.  `Workflow Orchestrators` like `ploomber` & `prefect` have gained in popularity for their ability to alert on failure.

If you don't find out about failure quickly your users will.

<br>

---
<br>