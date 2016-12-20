
<!-- README.md is generated from README.Rmd. Please edit that file -->
<!--
[![Build Status](https://travis-ci.org/hrbrmstr/sergeant.svg)](https://travis-ci.org/hrbrmstr/sergeant) 
![Project Status: Concept - Minimal or no implementation has been done yet.](http://www.repostatus.org/badges/0.1.0/concept.svg)](http://www.repostatus.org/#concept)
[![CRAN_Status_Badge](http://www.r-pkg.org/badges/version/sergeant)](http://cran.r-project.org/web/packages/sergeant) 
![downloads](http://cranlogs.r-pkg.org/badges/grand-total/sergeant)
-->
<img src="sergeant.png" width="33" align="left" style="padding-right:20px"/>

`sergeant` : Tools to Transform and Query Data with the 'Apache Drill' 'REST API' and JDBC Interface

Drill + `sergeant` is (IMO) a nice alternative to Spark + `sparklyr` if you don't need the ML components of Spark (i.e. just need to query "big data" sources, need to interface with parquet, need to combine disperate data source types — json, csv, parquet, rdbms - for aggregation, etc). Drill also has support for spatial queries.

The package doesn't have a `dplyr`-esque interface yet, but creating one is possible since Drill uses pretty standard SQL for queries. Right now, you need to build Drill SQL queries by hand and issue them with `drill_query()`. It's good to get one's hands dirty with some SQL on occassion (it builds character). The JDBC interface may make it possible to write a `dplyr` wrapper for it.

I find writing SQL queries to parquet files with Drill on a local 64GB Linux workstation to be more performant than doing the data ingestion work with R (for large or disperate data sets). I also work with many tiny JSON files on a daily basis and Drill makes it much easier to do so. YMMV.

You can download Drill from <https://drill.apache.org/download/> (use "Direct File Download"). I use `/usr/local/drill` as the install directory. `drill-embedded` is a super-easy way to get started playing with Drill on a single workstation and most of my workflows can get by using Drill this way. If there is sufficient desire for an automated downloader and a way to start the `drill-embedded` server from within R, please file an issue.

There are a few convenience wrappers for various informational SQL queries (like `drill_version()`). Please file an PR if you add more.

The package has been written with retrieval of rectangular data sources in mind. If you need/want a version of `drill_query()` that will enable returning of non-rectangular data (which is possible with Drill) then please file an issue.

Some of the more "controlling vs data ops" REST API functions aren't implemented. Please file a PR if you need those.

Finally, I run most of this locally and at home, so it's all been coded with no authentication or encryption in mind. If you want/need support for that, please file an issue. If there is demand for this, it will change the R API a bit (I've already thought out what to do but have no need for it right now).

The following functions are implemented:

-   `drill_connection`: Setup parameters for a Drill server/cluster connection
-   `drill_active`: Test whether Drill HTTP REST API server is up
-   `drill_cancel`: Cancel the query that has the given queryid
-   `drill_jdbc`: Connect to Drill using JDBC *(driver included with package until CRAN release)*
-   `drill_metrics`: Get the current memory metrics
-   `drill_options`: List the name, default, and data type of the system and session options
-   `drill_profile`: Get the profile of the query that has the given query id
-   `drill_profiles`: Get the profiles of running and completed queries
-   `drill_query`: Submit a query and return results
-   `drill_set`: Set Drill SYSTEM or SESSION options
-   `drill_settings_reset`: Changes (optionally, all) session settings back to system defaults
-   `drill_show_files`: Show files in a file system schema.
-   `drill_show_schemas`: Returns a list of available schemas.
-   `drill_stats`: Get Drillbit information, such as ports numbers
-   `drill_status`: Get the status of Drill
-   `drill_storage`: Get the list of storage plugin names and configurations
-   `drill_system_reset`: Changes (optionally, all) system settings back to system defaults
-   `drill_threads`: Get information about threads
-   `drill_uplift`: Turn a columnar query results into a type-converted tbl
-   `drill_use`: Change to a particular schema.
-   `drill_version`: Identify the version of Drill running

### Installation

``` r
devtools::install_github("hrbrmstr/sergeant")
```

### Experimental `dplyr` interface

``` r
library(RJDBC)
library(dplyr)
library(sergeant)

ds <- src_drill("localhost:31010", use_zk=FALSE)

ds
#> src:  Version: 1.9.0; Direct memory: 9,663,676,416 bytes
#> tbls: INFORMATION_SCHEMA, cp.default, dfs.default, dfs.root, dfs.tmp, sys

db <- tbl(ds, "cp.`employee.json`") 

# without `collect()`:
count(db, gender, marital_status)
#> Source:   query [?? x 3]
#> Database: Version: 1.9.0; Direct memory: 9,663,676,416 bytes
#> Groups: gender
#> 
#>   gender marital_status     n
#>    <chr>          <chr> <dbl>
#> 1      F              S   297
#> 2      M              M   278
#> 3      M              S   276
#> 4      F              M   304

# gets translated to:
# 
# SELECT *
# FROM (SELECT  gender ,  marital_status , COUNT(*) AS  n 
#       FROM  cp.`employee.json` 
#       GROUP BY  gender ,  marital_status )  govketbhqb 
# LIMIT 1000

count(db, gender, marital_status) %>% collect()
#> Source: local data frame [4 x 3]
#> Groups: gender [2]
#> 
#>   gender marital_status     n
#> *  <chr>          <chr> <dbl>
#> 1      F              S   297
#> 2      M              M   278
#> 3      M              S   276
#> 4      F              M   304

# gets translated to:
# 
# SELECT  gender ,  marital_status , COUNT(*) AS  n 
# FROM  cp.`employee.json` 
# GROUP BY  gender ,  marital_status 

group_by(db, position_title) %>% 
  count(gender) -> tmp2

group_by(db, position_title) %>% 
  count(gender) %>% 
  ungroup() %>% 
  mutate(full_desc=ifelse(gender=="F", "Female", "Male")) %>% 
  collect() %>% 
  select(Title=position_title, Gender=full_desc, Count=n)
#> # A tibble: 30 × 3
#>                     Title Gender Count
#> *                   <chr>  <chr> <dbl>
#> 1               President Female     1
#> 2      VP Country Manager   Male     3
#> 3      VP Country Manager Female     3
#> 4  VP Information Systems Female     1
#> 5      VP Human Resources Female     1
#> 6           Store Manager Female    13
#> 7              VP Finance   Male     1
#> 8           Store Manager   Male    11
#> 9            HQ Marketing Female     2
#> 10 HQ Information Systems Female     4
#> # ... with 20 more rows

# ^^ gets translated to:
# 
# SELECT  position_title ,  gender ,  n ,
#         CASE WHEN ( gender  = 'F') THEN ('Female') ELSE ('Male') END AS  full_desc 
# FROM (SELECT  position_title ,  gender , COUNT(*) AS  n 
#       FROM  cp.`employee.json` 
#       GROUP BY  position_title ,  gender )  dcyuypuypb 

arrange(db, desc(employee_id)) %>% print(n=20)
#> Source:   query [?? x 16]
#> Database: Version: 1.9.0; Direct memory: 9,663,676,416 bytes
#> 
#>    employee_id          full_name first_name last_name position_id          position_title store_id department_id
#>          <chr>              <chr>      <chr>     <chr>       <chr>                   <chr>    <chr>         <chr>
#> 1          999    Beverly Dittmar    Beverly   Dittmar          17 Store Permanent Stocker        8            17
#> 2          998  Elizabeth Jantzer  Elizabeth   Jantzer          17 Store Permanent Stocker        8            17
#> 3          997         John Sweet       John     Sweet          17 Store Permanent Stocker        8            17
#> 4          996     William Murphy    William    Murphy          17 Store Permanent Stocker        8            17
#> 5          995      Carol Lindsay      Carol   Lindsay          17 Store Permanent Stocker        8            17
#> 6          994      Richard Burke    Richard     Burke          17 Store Permanent Stocker        8            17
#> 7          993      Ethan Bunosky      Ethan   Bunosky          17 Store Permanent Stocker        8            17
#> 8          992  Claudette Cabrera  Claudette   Cabrera          17 Store Permanent Stocker        8            17
#> 9          991        Maria Terry      Maria     Terry          17 Store Permanent Stocker        8            17
#> 10         990        Stacey Case     Stacey      Case          17 Store Permanent Stocker        8            17
#> 11          99    Elizabeth Horne  Elizabeth     Horne          18 Store Temporary Stocker        6            18
#> 12         989    Dominick Nutter   Dominick    Nutter          17 Store Permanent Stocker        8            17
#> 13         988    Brian Willeford      Brian Willeford          17 Store Permanent Stocker        8            17
#> 14         987 Margaret Clendenen   Margaret Clendenen          17 Store Permanent Stocker        8            17
#> 15         986         Maeve Wall      Maeve      Wall          17 Store Permanent Stocker        8            17
#> 16         985     Mildred Morrow    Mildred    Morrow          16 Store Temporary Checker        8            16
#> 17         984      French Wilson     French    Wilson          16 Store Temporary Checker        8            16
#> 18         983   Elisabeth Duncan  Elisabeth    Duncan          16 Store Temporary Checker        8            16
#> 19         982     Linda Anderson      Linda  Anderson          16 Store Temporary Checker        8            16
#> 20         981      Selene Watson     Selene    Watson          16 Store Temporary Checker        8            16
#> # ... with more rows, and 8 more variables: birth_date <chr>, hire_date <chr>, salary <chr>, supervisor_id <chr>,
#> #   education_level <chr>, marital_status <chr>, gender <chr>, management_role <chr>

# ^^ gets translated to:
# 
# SELECT *
# FROM (SELECT *
#       FROM  cp.`employee.json` 
#       ORDER BY  employee_id  DESC)  lvpxoaejbc 
# LIMIT 5

db2 <- tbl(ds, "dfs.tmp.`/in/c.parquet`")
db2
#> Source:   query [?? x 3]
#> Database: Version: 1.9.0; Direct memory: 9,663,676,416 bytes
#> 
#>                  car   mpg   cyl
#>                <chr> <dbl> <chr>
#> 1          Mazda RX4  21.0     6
#> 2      Mazda RX4 Wag  21.0     6
#> 3         Datsun 710  22.8     4
#> 4     Hornet 4 Drive  21.4     6
#> 5  Hornet Sportabout  18.7     8
#> 6            Valiant  18.1     6
#> 7         Duster 360  14.3     8
#> 8          Merc 240D  24.4     4
#> 9           Merc 230  22.8     4
#> 10          Merc 280  19.2     6
#> # ... with more rows

db3 <- tbl(ds, "dfs.tmp.`/in/b.json`")
db3
#> Source:   query [?? x 3]
#> Database: Version: 1.9.0; Direct memory: 9,663,676,416 bytes
#> 
#>                  car  disp    wt
#>                <chr> <chr> <chr>
#> 1          Mazda RX4   160  2.62
#> 2      Mazda RX4 Wag   160 2.875
#> 3         Datsun 710   108  2.32
#> 4     Hornet 4 Drive   258 3.215
#> 5  Hornet Sportabout   360  3.44
#> 6            Valiant   225  3.46
#> 7         Duster 360   360  3.57
#> 8          Merc 240D 146.7  3.19
#> 9           Merc 230 140.8  3.15
#> 10          Merc 280 167.6  3.44
#> # ... with more rows

left_join(db2, db3)
#> Source:   query [?? x 5]
#> Database: Version: 1.9.0; Direct memory: 9,663,676,416 bytes
#> 
#>                  car   mpg   cyl              car0  disp    wt
#>                <chr> <dbl> <chr>             <chr> <chr> <chr>
#> 1          Mazda RX4  21.0     6         Mazda RX4   160  2.62
#> 2      Mazda RX4 Wag  21.0     6     Mazda RX4 Wag   160 2.875
#> 3         Datsun 710  22.8     4        Datsun 710   108  2.32
#> 4     Hornet 4 Drive  21.4     6    Hornet 4 Drive   258 3.215
#> 5  Hornet Sportabout  18.7     8 Hornet Sportabout   360  3.44
#> 6            Valiant  18.1     6           Valiant   225  3.46
#> 7         Duster 360  14.3     8        Duster 360   360  3.57
#> 8          Merc 240D  24.4     4         Merc 240D 146.7  3.19
#> 9           Merc 230  22.8     4          Merc 230 140.8  3.15
#> 10          Merc 280  19.2     6          Merc 280 167.6  3.44
#> # ... with more rows

# ^^ gets translated to:
# 
# SELECT *
# FROM (SELECT * FROM  dfs.tmp.`/in/c.parquet` 
#       LEFT JOIN dfs.tmp.`/in/b.json` 
#       USING ( car ))  gnyhbahqil 
# LIMIT 1000
```

### Usage

``` r
library(sergeant)

# current verison
packageVersion("sergeant")
#> [1] '0.2.0.9000'

dc <- drill_connection("localhost") 

drill_active(dc)
#> [1] TRUE

drill_version(dc)
#> [1] "1.9.0"

drill_storage(dc)$name
#> [1] "cp"    "dfs"   "hbase" "hive"  "kudu"  "mongo" "s3"
```

Working with the built-in JSON data sets:

``` r
drill_query(dc, "SELECT * FROM cp.`employee.json` limit 100")
#> Parsed with column specification:
#> cols(
#>   store_id = col_integer(),
#>   gender = col_character(),
#>   department_id = col_integer(),
#>   birth_date = col_date(format = ""),
#>   supervisor_id = col_integer(),
#>   last_name = col_character(),
#>   position_title = col_character(),
#>   hire_date = col_datetime(format = ""),
#>   management_role = col_character(),
#>   salary = col_double(),
#>   marital_status = col_character(),
#>   full_name = col_character(),
#>   employee_id = col_integer(),
#>   education_level = col_character(),
#>   first_name = col_character(),
#>   position_id = col_integer()
#> )
#> # A tibble: 100 × 16
#>    store_id gender department_id birth_date supervisor_id last_name         position_title  hire_date   management_role
#> *     <int>  <chr>         <int>     <date>         <int>     <chr>                  <chr>     <dttm>             <chr>
#> 1         0      F             1 1961-08-26             0    Nowmer              President 1994-12-01 Senior Management
#> 2         0      M             1 1915-07-03             1   Whelply     VP Country Manager 1994-12-01 Senior Management
#> 3         0      M             1 1969-06-20             1    Spence     VP Country Manager 1998-01-01 Senior Management
#> 4         0      F             1 1951-05-10             1 Gutierrez     VP Country Manager 1998-01-01 Senior Management
#> 5         0      F             2 1942-10-08             1   Damstra VP Information Systems 1994-12-01 Senior Management
#> 6         0      F             3 1949-03-27             1  Kanagaki     VP Human Resources 1994-12-01 Senior Management
#> 7         9      F            11 1922-08-10             5   Brunner          Store Manager 1998-01-01  Store Management
#> 8        21      F            11 1979-06-23             5  Blumberg          Store Manager 1998-01-01  Store Management
#> 9         0      M             5 1949-08-26             1     Stanz             VP Finance 1994-12-01 Senior Management
#> 10        1      M            11 1967-06-20             5  Murraiin          Store Manager 1998-01-01  Store Management
#> # ... with 90 more rows, and 7 more variables: salary <dbl>, marital_status <chr>, full_name <chr>, employee_id <int>,
#> #   education_level <chr>, first_name <chr>, position_id <int>

drill_query(dc, "SELECT COUNT(gender) AS gender FROM cp.`employee.json` GROUP BY gender")
#> Parsed with column specification:
#> cols(
#>   gender = col_integer()
#> )
#> # A tibble: 2 × 1
#>   gender
#> *  <int>
#> 1    601
#> 2    554

drill_options(dc)
#> # A tibble: 105 × 4
#>                                              name value   type    kind
#> *                                           <chr> <chr>  <chr>   <chr>
#> 1                  planner.enable_hash_single_key  TRUE SYSTEM BOOLEAN
#> 2              planner.enable_limit0_optimization FALSE SYSTEM BOOLEAN
#> 3               store.json.read_numbers_as_double FALSE SYSTEM BOOLEAN
#> 4                 planner.enable_constant_folding  TRUE SYSTEM BOOLEAN
#> 5                       store.json.extended_types FALSE SYSTEM BOOLEAN
#> 6    planner.memory.non_blocking_operators_memory    64 SYSTEM    LONG
#> 7                   planner.enable_multiphase_agg  TRUE SYSTEM BOOLEAN
#> 8  planner.filter.max_selectivity_estimate_factor     1 SYSTEM  DOUBLE
#> 9                     planner.enable_mux_exchange  TRUE SYSTEM BOOLEAN
#> 10                   store.parquet.use_new_reader FALSE SYSTEM BOOLEAN
#> # ... with 95 more rows

drill_options(dc, "json")
#> # A tibble: 7 × 4
#>                                                    name value   type    kind
#>                                                   <chr> <chr>  <chr>   <chr>
#> 1                     store.json.read_numbers_as_double FALSE SYSTEM BOOLEAN
#> 2                             store.json.extended_types FALSE SYSTEM BOOLEAN
#> 3                              store.json.writer.uglify FALSE SYSTEM BOOLEAN
#> 4                store.json.reader.skip_invalid_records FALSE SYSTEM BOOLEAN
#> 5 store.json.reader.print_skipped_invalid_record_number FALSE SYSTEM BOOLEAN
#> 6                              store.json.all_text_mode  TRUE SYSTEM BOOLEAN
#> 7                    store.json.writer.skip_null_fields  TRUE SYSTEM BOOLEAN
```

Working with parquet files
--------------------------

``` r
drill_query(dc, "SELECT * FROM dfs.`/usr/local/drill/sample-data/nation.parquet` LIMIT 5")
#> Parsed with column specification:
#> cols(
#>   N_COMMENT = col_character(),
#>   N_NAME = col_character(),
#>   N_NATIONKEY = col_integer(),
#>   N_REGIONKEY = col_integer()
#> )
#> # A tibble: 5 × 4
#>              N_COMMENT    N_NAME N_NATIONKEY N_REGIONKEY
#> *                <chr>     <chr>       <int>       <int>
#> 1  haggle. carefully f   ALGERIA           0           0
#> 2 al foxes promise sly ARGENTINA           1           1
#> 3 y alongside of the p    BRAZIL           2           1
#> 4 eas hang ironic, sil    CANADA           3           1
#> 5 y above the carefull     EGYPT           4           4
```

Including multiple parquet files in different directories (note the wildcard support):

``` r
drill_query(dc, "SELECT * FROM dfs.`/usr/local/drill/sample-data/nations*/nations*.parquet` LIMIT 5")
#> Parsed with column specification:
#> cols(
#>   N_COMMENT = col_character(),
#>   N_NAME = col_character(),
#>   N_NATIONKEY = col_integer(),
#>   N_REGIONKEY = col_integer(),
#>   dir0 = col_character()
#> )
#> # A tibble: 5 × 5
#>              N_COMMENT    N_NAME N_NATIONKEY N_REGIONKEY      dir0
#> *                <chr>     <chr>       <int>       <int>     <chr>
#> 1  haggle. carefully f   ALGERIA           0           0 nationsMF
#> 2 al foxes promise sly ARGENTINA           1           1 nationsMF
#> 3 y alongside of the p    BRAZIL           2           1 nationsMF
#> 4 eas hang ironic, sil    CANADA           3           1 nationsMF
#> 5 y above the carefull     EGYPT           4           4 nationsMF
```

### A preview of the built-in support for spatial ops

Via: <https://github.com/k255/drill-gis>

A common use case is to select data within boundary of given polygon:

``` r
drill_query(dc, "
select columns[2] as city, columns[4] as lon, columns[3] as lat
    from cp.`sample-data/CA-cities.csv`
    where
        ST_Within(
            ST_Point(columns[4], columns[3]),
            ST_GeomFromText(
                'POLYGON((-121.95 37.28, -121.94 37.35, -121.84 37.35, -121.84 37.28, -121.95 37.28))'
                )
            )
")
#> Parsed with column specification:
#> cols(
#>   city = col_character(),
#>   lon = col_double(),
#>   lat = col_double()
#> )
#> # A tibble: 7 × 3
#>          city       lon      lat
#> *       <chr>     <dbl>    <dbl>
#> 1     Burbank -121.9316 37.32328
#> 2    San Jose -121.8950 37.33939
#> 3        Lick -121.8458 37.28716
#> 4 Willow Glen -121.8897 37.30855
#> 5 Buena Vista -121.9166 37.32133
#> 6    Parkmoor -121.9308 37.32105
#> 7   Fruitdale -121.9327 37.31086
```

### JDBC

``` r
library(RJDBC)

con <- drill_jdbc("localhost:31010", use_zk=FALSE)
#> Using [jdbc:drill:drillbit=localhost:31010]...

drill_query(con, "SELECT * FROM cp.`employee.json`")
#> # A tibble: 1,155 × 16
#>    employee_id         full_name first_name last_name position_id         position_title store_id department_id
#> *        <chr>             <chr>      <chr>     <chr>       <chr>                  <chr>    <chr>         <chr>
#> 1            1      Sheri Nowmer      Sheri    Nowmer           1              President        0             1
#> 2            2   Derrick Whelply    Derrick   Whelply           2     VP Country Manager        0             1
#> 3            4    Michael Spence    Michael    Spence           2     VP Country Manager        0             1
#> 4            5    Maya Gutierrez       Maya Gutierrez           2     VP Country Manager        0             1
#> 5            6   Roberta Damstra    Roberta   Damstra           3 VP Information Systems        0             2
#> 6            7  Rebecca Kanagaki    Rebecca  Kanagaki           4     VP Human Resources        0             3
#> 7            8       Kim Brunner        Kim   Brunner          11          Store Manager        9            11
#> 8            9   Brenda Blumberg     Brenda  Blumberg          11          Store Manager       21            11
#> 9           10      Darren Stanz     Darren     Stanz           5             VP Finance        0             5
#> 10          11 Jonathan Murraiin   Jonathan  Murraiin          11          Store Manager        1            11
#> # ... with 1,145 more rows, and 8 more variables: birth_date <chr>, hire_date <chr>, salary <chr>, supervisor_id <chr>,
#> #   education_level <chr>, marital_status <chr>, gender <chr>, management_role <chr>

# but it can work via JDBC function calls, too
dbGetQuery(con, "SELECT * FROM cp.`employee.json`") %>% 
  tibble::as_tibble()
#> # A tibble: 1,155 × 16
#>    employee_id         full_name first_name last_name position_id         position_title store_id department_id
#> *        <chr>             <chr>      <chr>     <chr>       <chr>                  <chr>    <chr>         <chr>
#> 1            1      Sheri Nowmer      Sheri    Nowmer           1              President        0             1
#> 2            2   Derrick Whelply    Derrick   Whelply           2     VP Country Manager        0             1
#> 3            4    Michael Spence    Michael    Spence           2     VP Country Manager        0             1
#> 4            5    Maya Gutierrez       Maya Gutierrez           2     VP Country Manager        0             1
#> 5            6   Roberta Damstra    Roberta   Damstra           3 VP Information Systems        0             2
#> 6            7  Rebecca Kanagaki    Rebecca  Kanagaki           4     VP Human Resources        0             3
#> 7            8       Kim Brunner        Kim   Brunner          11          Store Manager        9            11
#> 8            9   Brenda Blumberg     Brenda  Blumberg          11          Store Manager       21            11
#> 9           10      Darren Stanz     Darren     Stanz           5             VP Finance        0             5
#> 10          11 Jonathan Murraiin   Jonathan  Murraiin          11          Store Manager        1            11
#> # ... with 1,145 more rows, and 8 more variables: birth_date <chr>, hire_date <chr>, salary <chr>, supervisor_id <chr>,
#> #   education_level <chr>, marital_status <chr>, gender <chr>, management_role <chr>
```

### Use in knitr SQL code chunks

If you install `knit` via GitHub (`devtools::install_github("yihui/knitr")) you can use the`sql`chunk code type with`drill\_jdbc()\` connections:

    ---
    output: html_document
    ---


    ```r
    library(sergeant)
    library(DBI)
    library(RJDBC)
    ```

    ## Setup JDBC connection


    ```r
    dc <- drill_jdbc("localhost:31010", use_zk=FALSE)
    #> Using [jdbc:drill:drillbit=localhost:31010]...
    ```

    ## Test out a query


    ```sql
    SELECT * FROM cp.`employee.json`
    ```


    <div class="knitsql-table">


    Table: Displaying records 1 - 10

    employee_id   full_name           first_name   last_name   position_id   position_title           store_id   department_id   birth_date   hire_date               salary       supervisor_id   education_level    marital_status   gender   management_role   
    ------------  ------------------  -----------  ----------  ------------  -----------------------  ---------  --------------  -----------  ----------------------  -----------  --------------  -----------------  ---------------  -------  ------------------
    1             Sheri Nowmer        Sheri        Nowmer      1             President                0          1               1961-08-26   1994-12-01 00:00:00.0   80000.0000   0               Graduate Degree    S                F        Senior Management 
    2             Derrick Whelply     Derrick      Whelply     2             VP Country Manager       0          1               1915-07-03   1994-12-01 00:00:00.0   40000.0000   1               Graduate Degree    M                M        Senior Management 
    4             Michael Spence      Michael      Spence      2             VP Country Manager       0          1               1969-06-20   1998-01-01 00:00:00.0   40000.0000   1               Graduate Degree    S                M        Senior Management 
    5             Maya Gutierrez      Maya         Gutierrez   2             VP Country Manager       0          1               1951-05-10   1998-01-01 00:00:00.0   35000.0000   1               Bachelors Degree   M                F        Senior Management 
    6             Roberta Damstra     Roberta      Damstra     3             VP Information Systems   0          2               1942-10-08   1994-12-01 00:00:00.0   25000.0000   1               Bachelors Degree   M                F        Senior Management 
    7             Rebecca Kanagaki    Rebecca      Kanagaki    4             VP Human Resources       0          3               1949-03-27   1994-12-01 00:00:00.0   15000.0000   1               Bachelors Degree   M                F        Senior Management 
    8             Kim Brunner         Kim          Brunner     11            Store Manager            9          11              1922-08-10   1998-01-01 00:00:00.0   10000.0000   5               Bachelors Degree   S                F        Store Management  
    9             Brenda Blumberg     Brenda       Blumberg    11            Store Manager            21         11              1979-06-23   1998-01-01 00:00:00.0   17000.0000   5               Graduate Degree    M                F        Store Management  
    10            Darren Stanz        Darren       Stanz       5             VP Finance               0          5               1949-08-26   1994-12-01 00:00:00.0   50000.0000   1               Partial College    M                M        Senior Management 
    11            Jonathan Murraiin   Jonathan     Murraiin    11            Store Manager            1          11              1967-06-20   1998-01-01 00:00:00.0   15000.0000   5               Graduate Degree    S                M        Store Management  

    </div>

Which is (IMO) *way* better than using the Drill consoles, the Drill Web UI query box or SQLWorkbench.

### Test Results

``` r
library(sergeant)
library(testthat)
#> 
#> Attaching package: 'testthat'
#> The following object is masked from 'package:dplyr':
#> 
#>     matches

date()
#> [1] "Mon Dec 19 19:58:29 2016"

test_dir("tests/")
#> testthat results ========================================================================================================
#> OK: 0 SKIPPED: 0 FAILED: 0
#> 
#> DONE ===================================================================================================================
```

### Code of Conduct

Please note that this project is released with a [Contributor Code of Conduct](CONDUCT.md). By participating in this project you agree to abide by its terms.
