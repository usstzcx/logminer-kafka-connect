# Logminer Kafka Connect

Logminer Kafka Connect is a CDC Kafka Connect source for Oracle Databases (tested with Oracle 11.2.0.4). 

Changes are extracted from the Archivelog using [Oracle Logminer](https://docs.oracle.com/cd/B19306_01/server.102/b14215/logminer.htm). 

Currently supported features:
- Insert, Update and Delete changes will be tracked on configured tables
- Logminer without "CONTINUOUS_MINE", thus in theory being compatible with Oracle 19c (not tested)
- Initial import of the current table state
- Only based on Oracle features that are available in Oracle XE (and thus available in all Oracle versions). No
Oracle GoldenGate license required!

Planned features:
- More documentation :)
- Reading schema changes from the Archive-Log. Currently the online catalog is used. See 
https://docs.oracle.com/cd/B19306_01/server.102/b14215/logminer.htm#i1014687 for more information.


## Initial Import
If the start SCN is not set or set to 0, logminer kafka connect will query
the configured tables for an initial import. During the initial import, no
DDL statements should be executed against the database. Otherwise the initial import will
fail.

To support initial import, database flashback queries need to be enabled on the source database.

All rows that are in the table at time of initial import will be treated as "INSERT" changes.

## Change Types
The change types are compatible to change types published by the debezium (https://debezium.io/) project.
Thus it is easy to migrate to the official debezium Oracle plugin ones it reaches a stable state.
The key of the kafka topic will be filled with a struct containing the primary key values of the changed row. 

### Value Struct
The value is a structure having the following fields:
  -  `op`  
      Operation that has been executed.
        - Type: string
        - Possible values:
            - 'r' - Read on initial import
            - 'i' - Insert
            - 'u' - Update
            - 'd' - Delete
  - `before`
     Image of the row before the operation has been executed. Contains all columns.
       - Type: struct
       - Only filled for the following operations:
           - Update
           - Delete
  - `after`
     Image of the row after the operation has been executed. Contains all columns.
       - Type: struct
       - Only filled for the following operations:
           - Insert
           - Read
           - Update
  - `ts_ms`
     Timestamp of import as millis since epoch
       - Type: Long
  - `source`
     Additional information about this change record from the source database. 
       - Type: source

### Source struct       

The following source fields will be provided:
  - `version`  
     Version of this component
       - Type: string
       - Value: '1.0'
  - `connector`  
     Name of this connector.
       - Type: string
       - Value: 'logminer-kafka-connect'
  - `ts_ms`  
     Timestamp of the change in the source database.
       - Type: long
       - Logical Name: org.apache.kafka.connect.data.Timestamp
  - `scn`  
     SCN number of the change. For the initial import this is the scn of the las update to the row.
       - Type: long
  - `txId`  
     Transaction in which the change occurred in the source database. For the initial import, this field is always null.
       - Type: string (optional)       
  - `table`  
     Table in the source database for which the change was recorded.
       - Type: string 
  - `schema`  
     Schema in the source database in which the change was recorded.
      - Type: string
  - `user`  
     The user that triggered the change
       - Type: string (optional)


## Configuration
  - `db.hostname`  
    Database hostname
    
      - Type: string
      - Importance: high

  - `db.name`  
    Database SID
    
      - Type: string
      - Importance: high

  - `db.port`  
    Database port (usually 1521)
    
      - Type: int
      - Importance: high

  - `db.user`  
    Database user
    
      - Type: string
      - Importance: high

  - `db.user.password`  
    Database password
    
      - Type: string
      - Importance: high

  - `batch.size`  
    Batch size of rows that should be fetched in one batch
    
      - Type: int
      - Default: 1000
      - Importance: high

  - `record.prefix`  
    Prefix of the subject record. If you're using an Avro converter,
    this will be the namespace.
    
      - Type: string
      - Default: ""
      - Importance: high

  - `table.whitelist`  
    Tables that should be monitored, separated by ','. Tables have to be
    specified with schema. You can also justspecify a schema to indicate
    that all tables within that schema should be monitored. Examples:
    'MY\_USER.TABLE, OTHER\_SCHEMA'.
    
      - Type: string
      - Default: ""
      - Importance: high

  - `topic.prefix`  
    Prefix for the topic. Each monitored table will be written to a
    separate topic. If you want to changethis behaviour, you can add a
    RegexRouter transform.
    
      - Type: string
      - Default: ""
      - Importance: medium

  - `db.fetch.size`  
    JDBC result set prefetch size. If not set, it will be defaulted to
    batch.size. The fetch should not be smaller than the batch size.
    
      - Type: int
      - Default: null
      - Importance: low

  - `start.scn`  
    Start SCN, if set to 0 an initial intake from the tables will be
    performed.
    
      - Type: long
      - Default: 0
      - Importance: low


