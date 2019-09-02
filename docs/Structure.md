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
{indexDatumPrefix}{tableId}{indexId}{lookupLength}{lookupItemLength}{lookupItem}{primaryKeyEncoded}
^_________________^________^________^_____________^_________________^___________^__________________
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

This format does not allow for a partial match on a single column. Meaing
no LIKE queries. But any query that is doing a comparison this index format
should be sufficient to do a prefix scan as long as the query planner scans
in the order that the columns are indexed (regardless of the order they are
filtered in the query).

# Datum

A datum is the actual cell of data that is stored, it is the representation
of a row/column pair.

```
{datumPrefix}{tableId}{columnId}{primaryKeyEncoded}
```

The datum stores the value of the cell as the value of the key value pair.
The datum also has the encoded primary key as the suffix unlike CockroachDB.
In CockroachDB the columnId is the suffix, which allows for very fast
retrieval for all of the colunms for a single primary key. However I would
like to optimize for not retrieving every column for most queries, so by
flipping the two then I can retrieve specific columns much faster with
a prefix scan. I could iterate over an entire column for a table with
a single prefix

```
SELECT id FROM products;
```

In CockroachDB the query planner would have two options for this query,
1. Do a prefix scan over the entire table regardless of primary key.
Essentially retrieving every column in the table but throwing away anything
except for the id column.
2. Do a prefix scan over an index that stores or has an early lookup for
the id column.

But with the datum format that I'm using I could do:
1. A prefix scan over all datums for the given table and column regardless
of an index or primary key.

For the primaryKey suffix datum format it would always be more efficient
to do a table scan than doing an index scan for that one column. An index
key will always be larger than the datum key itself in this case.

This dynamic changes when there are 2 columns. For Cockroach the plan
stays the same; either iterate all the things or find an index that
has the columns you need stored or in the lookup that you can use to
return results without a table scan.

But in my format the plan stays the same but another iteration is
performed, and it could potentially be performed with the same iterator.

The first iteration would retrieve all of the values for the first column,
and the second iteration would retrieve all of the values for the second column.

Without indexes my datum format should technically outperform CockroachDB
with an identical data set.

I will need to test this to be sure as I may be missing something in
my understanding of CockroachDB's storage architecture.

