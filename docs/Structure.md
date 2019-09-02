# Structure

This document outlines the very base structure of a key-value based SQL
database.

# Tables

Tables are represented by a single key value pair in the following format.

```
{tablePrefix}{tableNameLength}{tableName}{tableId}
^____________^________________^__________^________
|            |                |          |
1 Byte       |                |          |
             |                |          |
             1 Byte           |          |
                              |          |
                              <= 255 Bytes
                                         |
                                         1 Byte
```

The maximum size for a table key is 258 bytes encoded. This could
be made potentially smaller by removing the tableId suffix and making
the value of the pair the tableId instead.

This design limits a single database to 255 different tables. This
should be more than enough for most use cases and allows key sizes
for prefix scans to be as small as possible.

Besides potentially storing the tableId as the value of the pair there
is not currently any other data that would be stored on the value of
a table record. Most of the tables metadata is stored as separate
records.

# Columns

Columns are stored as a single key value pair as well but have some
data stored on the value.

```
{columnPrefix}{tableId}{columnNameLength}{columnName}{columnId}
^_____________^________^_________________^___________^_________
|             |        |                 |           |
1 Byte        |        |                 |           |
              |        |                 |           |
              1 Byte   |                 |           |
                       |                 |           |
                       1 Byte            |           |
                                         |           |
                                         <= 255 Bytes|
                                                     |
                                                     1 Byte
```

The maximum number of columns that a single table could have is 255.
This number is pretty much the maximum for everything except for rows
in the table. 

ColumnIds are numeric starting at 1 based on the order of the columns
defined in the CREATE TABLE query.

Column values should store the following metadata:

- Primary Key?
- Serial Column
- Column type
- Indexed
- Index IDs
- Unique
- Unique Constraint IDs

# Indexes

Columns can have indexes, that is to say that; multiple columns can have
a single index or a single column can have multiple indexes. Indexes are
stored in the database as they are declared. If an index has lookup
columns a, b and c. Then the encoded lookup value will be a then b then
c. All indexes are stored sorted due to the nature of the key value store.
You cannot change the direction an index is sorted, if an index needs to
be iterated in an ASC or DESC direction then the key value store will
reverse the iteration as needed.

The query planner should try to select indexes that have a high cardinality
over indexes that do not. If a query is filtering by multiple columns
the query planner should sort those filters with the following criteria.

- Primary keys.
- Columns with indexes.
- Columns with the highest cardinality. If a column has the potential to have
more unique values then it is ideal for searching for primary keys or narrowing
down other indexes.
    - Unique columns are perfect here and would involve a unique index lookup
    which is ideal given that it is a single lookup rather than a range scan.
- more later...

Indexes are stored as two separate records, the first being the index meta
record. The meta record stores the information about the index and what
columns are being stored and the order they are stored in.

I'm not sure yet what I want the meta data key and value to look like.
```
{indexMetaPrefix}{tableId}
```

The other record for an index is the index datum. The datum is the actual
key that is scanned when performing a query. 

```
{indexDatumPrefix}{tableId}{indexId}{lookupLength}{lookupItemLength}{lookupItem}{primaryKey}
^_________________^________^________^_____________^_________________^___________^___________
|                 |        |        |             |                 |           |
1 Byte            |        |        |             |                 |           |
                  |        |        |             |                 |           |
                  1 Byte   |        |             |                 |           |
                           |        |             |                 |           |
                           1 Byte   |             |                 |           |
                                    |             |                 |           |
                                    1 Byte        |                 |           |
                                                  |                 |           |
                                                  1 Byte Per Item   |           |
                                                                    |           |
                                                                    <= 255 Bytes Per Item
                                                                                |
                                                                                > 1 Byte (Encoded Primary Key)
                                                                                

```
