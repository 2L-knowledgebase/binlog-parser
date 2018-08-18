# binlog-parser

A tool for parsing a MySQL binlog file to JSON. Reads a binlog input file, queries a database for field names, writes JSON to stdout. The output looks like this:

```
# WRITE_ROWS_EVENT (Insert) 
{  
   "Header":{  
      "Schema":"feed",
      "Table":"ob_live",
      "BinlogMessageTime":"2018-08-18T08:26:22Z",
      "BinlogPosition":407,
      "XId":99
   },
   "Type":"Insert",
   "Data":{  
      "Row":{  
         "curr_status":1,
         "member":"mark",
         "pre_status":1,
         "update_time":"2018-08-18 16:26:22"
      },
      "MappingNotice":""
   }
}

# UPDATE_ROWS_EVENT
{  
   "Header":{  
      "Schema":"feed",
      "Table":"ob_live",
      "BinlogMessageTime":"2018-08-18T08:27:26Z",
      "BinlogPosition":981,
      "XId":102
   },
   "Type":"Update",
   "OldData":{  
      "Row":{  
         "curr_status":1,
         "member":"mark",
         "pre_status":1,
         "update_time":"2018-08-18 16:26:22"
      },
      "MappingNotice":""
   },
   "Data":{  
      "Row":{  
         "curr_status":1,
         "member":"mark",
         "pre_status":2,
         "update_time":"2018-08-18 16:26:22"
      },
      "MappingNotice":""
   }
}

# DELETE_ROWS_EVENT 
{  
   "Header":{  
      "Schema":"feed",
      "Table":"ob_live",
      "BinlogMessageTime":"2018-08-18T08:27:45Z",
      "BinlogPosition":1257,
      "XId":103
   },
   "Type":"Delete",
   "Data":{  
      "Row":{  
         "curr_status":1,
         "member":"eric",
         "pre_status":1,
         "update_time":"2018-08-18 16:26:39"
      },
      "MappingNotice":""
   }
}

```

# Installation

> Requires Go version 1.7 or higher.

```
$ git clone https://github.com/zalora/binlog-parser.git
$ cd binlog-parser
$ git submodule update --init

# since https://go.googlesource.com is blocked by chinese firewall
# use proxy e.g. ss, via https://github.com/shadowsocks/shadowsocks-windows/issues/407
$ HTTP_PROXY="http://127.0.0.1:1080" HTTPS_PROXY="http://127.0.0.1:1080" git submodule update --init
              
$ make        
$ ./bin/binlog-parser -h
```           

## Assumptions

- It is assumed that MySQL row-based binlog format is used (or mixed, but be aware, that then only the row-formatted data in mixed binlogs can be extracted)
- This tool is written with MySQL 5.6 in mind, although it should also work for MariaDB when GTIDs are not used

# Usage

Run `binlog-parser -h` to get the list of available options:

    Usage:	binlog-parser [options ...] binlog

    Options are:

      -alsologtostderr
        	log to standard error as well as files
      -include_schemas string
        	comma-separated list of schemas to include
      -include_tables string
        	comma-separated list of tables to include
      -log_backtrace_at value
        	when logging hits line file:N, emit a stack trace
      -log_dir string
        	If non-empty, write log files in this directory
      -logtostderr
        	log to standard error instead of files
      -prettyprint
        	Pretty print json
      -stderrthreshold value
        	logs at or above this threshold go to stderr
      -v value
        	log level for V logs
      -vmodule value
        	comma-separated list of pattern=N settings for file-filtered logging

    Required environment variables:

    DB_DSN	 Database connection string, needs read access to information_schema

## Example usage

Using `dbuser` and no password, connecting to `information_schema` database on localhost, parsing the binlog file `/some/binlog.bin`:

```
DB_DSN=dbuser@/information_schema ./binlog-parser /some/binlog.bin
```

## Matching field names and data

The mysql binlog format doesn't include the fieldnames for row events (INSERT/UPDATE/DELETE). As the goal of the parser is to output
usable JSON, it connects to a running MySQL instance and queries the `information_schema` database for information on field names in the table.

The database connection is creatd by using the environment variable `DB_DSN`, which should contain the database credentials in the form of
`user:password@/dbname` - the format that the [Go MySQL driver](https://godoc.org/github.com/go-sql-driver/mysql) uses.

```
export DB_DSN='<username>:<passwd>@<protocol>(<host>:<port>)/information_schema' ./bin/binlog-parser /some/binlog

# https://stackoverflow.com/questions/23550453/golang-how-to-open-a-remote-mysql-connection
e.g. export DB_DSN='root:123@tcp(localhost:3306)/information_schema' ./bin/binlog-parser /tmp/mysql-bin.000001
```

## Effect of schema changes

As this tool doesn't keep an internal representation of the database schema, it is very well possible that the database schema and the schema used in the
queries in the binlog file already have diverged (e. g. parsing a binlog file from a few days ago, but the schema on the main database already changed
by dropping or adding columns).

The parser will NOT make an attempt to map data to fields in a table if the information schema retuns more or too less columns
compared to the format found in the binlog. The field names will be mapped as "unknown":

    {
        "Header": {
            "Schema": "test_db",
            "Table": "employees",
            "BinlogMessageTime": "2017-04-13T08:02:04Z",
            "BinlogPosition": 635,
            "XId": 8
        },
        "Type": "Insert",
        "Data": {
            "Row": {
                "(unknown_0)": 1,
                "(unknown_1)": "2017-04-13",
                "(unknown_2)": "Max",
                "(unknown_3)": "Mustermann"
            },
            "MappingNotice": "column names array is missing field(s), will map them as unknown_*"
        }
    }

### More complex case

Changing the order of fields in a table can lead to unexpected parser results. Consider an example binlog file `A.bin`.
A query like `INSERT INTO db.foo SET field_1 = 10, field_2 = 20` will look in the binlog like this:

    ...
    ### INSERT INTO `db`.`foo`
    ### SET
    ###   @1=20 /* ... */
    ###   @2=20 /* ... */
    ...

The parser queries `information_schema` for the field names of the `db.foo` table:

    +-------------+-----+
    | Field       | ... |
    +-------------+-----+
    | field_1     | ... |
    | field_2     | ... |
    +-------------+-----+

The fields will be mapped by the parser in the order as specified in the table and the JSON will look like this:

    {
        ...
        "Type": "Insert",
        "Data": {
            "Row": {
                "field_1": 10,
                "field_2": 20
            }
        }
    }
    ...

Now if a schema change happened after some time, `db.foo` fields might look now like this (the order of the fiels changed):

    +-------------+-----+
    | Field       | ... |
    +-------------+-----+
    | field_2     | ... |
    | field_1     | ... |
    +-------------+-----+

If you parse the same binlog file `A.bin` now again, but against the new schema of `db.foo` (in which the fields changed position), the resulting JSON
will look like that:


    {
        ...
        "Type": "Insert",
        "Data": {
            "Row": {
                "field_2": 10,
                "field_1": 20
            }
        }
    }
    ...

This means you have to be very careful when parsing old binlog files, as the db schema can have evolved since the binlog was generated and the parser
has no way of knowing of these changes.

If this limitation is not acceptable, some tools like [Maxwell's Daemon by Zendesk](https://github.com/zendesk/maxwell) can work around that issue at the cost of greater complexity.

