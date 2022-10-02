
# FreeDB Protocols

## Row Store

Row Store is a data structure that stores records as a sequence of rows (similar to RDBMS like MySQL, PostgreSQL, etc).

### Data Structure

The underlying format that we're going to use to store the data in Google Sheets is fairly simple. The sheet will have
`N + 1` columns where `N` is the number of columns that the client specified in the schema. The extra 1 column will be
used to store `_rid` metadata (which will be explained later).

The first row will be used to store the column name of each column. The first column will be the `_rid` metadata column
(which also has the `_rid` column name) and will be followed by the first column in the data schema, then followed by the
second one, and so on. The next rows (second row and rows below that) will be used to store the records.

Suppose the client schema looks like this (written in `PyFreeDB` model):
```
class Person(models.Model):
    name = models.StringField() # 1st column
    age = models.IntegerField() # 2nd column
```

and we have data below:
```
[
    Person(name="A", age=1), # 1st row
    Person(name="B", age=2), # 2nd row
    Person(name="C", age=3), # 3rd row
    Person(name="D", age=4)  # 4th row
]
```

Then the actual data that we store in Google Sheet would look like this:
| _rid | name | age |
| ---- | ---- | --- |
| 1    | A    | 1   |
| 2    | B    | 2   |
| 3    | C    | 3   |
| 4    | D    | 4   |

#### About `_rid` Metadata

`_rid` column will be used to tell the index of a row. This metadata will make our Google Sheet Formula simpler
so that we can utilize [Google Visualization API][GVizAPI] for our needs.

⚠️ Even though `_rid` can be used as the row identifier (because `_rid` is unique between each row), we recommend the
client to implement their own row identifier mechanism such as [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier)
or [Snowflake](https://en.wikipedia.org/wiki/Snowflake_ID) because we will not guarantee that `_rid` will not be
reused out-of-the-box (to ensure the `_rid` is not reused, you need to implement soft-delete mechanism by your own).

Consider this scenario:
1. Client inserts Row A to the store, which will have `_rid=2`.
2. Client deletes Row A from the store.
3. Client Insert Row B to the store, which will reuse the same `_rid` as A i.e. B's `_rid` will be equal to `2`.

We did not expose this field by default because if the client is not aware of this behavior it can cause various bugs
because of the invalid/dangling IDs.

### Operations

#### Insert

Inserts new rows into the database.  The operation method should expect these parameters:
| Parameter Name | Required | Remarks                                                   |
| -------------- | -------- | --------------------------------------------------------- |
| `rows`         | Yes      | List of row objects that will be inserted into the store. |


Call [spreadsheets.values.append][AppendAPI]
API with the following parameters:

| Parameter Name              | Value                      |
| --------------------------- | -------------------------- |
| `spreadsheetId`             | `<current_spreadsheet_id>` |
| `range`                     | `<data_range>`             |
| `responseValueRenderOption` | `FORMATTED_VALUE`          |
| `valueInputOption`          | `client_ENTERED`           |
| `includeValuesInResponse`   | `true`                     |
| `insertDataOption`          | `OVERWRITE`                |

Replace the `<current_spreadsheet_id>` with the Google Spreadsheet ID that we're going to operate on and `<data_range>`
with the range where the data lives in [A1 notation format][A1Notation]. It's recommended to select the entire column
for the `<data_range>`, e.g. suppose we have 5 columns (including `_rid` metadata column) in `Sheet1` we should
use `Sheet1!A:E` to select the entire rows.

To construct the value of `values` parameter (inside the request's body), convert each row into an array where the
1st value (value of the `_rid` metadata column) equals to `"=ROW()"` and `i`-th (1-based index) value contains the
value of `i`-th column of the current row.

Suppose we want to insert the data below:
```
[
    Person(name="A", age=1), # 1st row
    Person(name="B", age=2), # 2nd row
    Person(name="C", age=3), # 3rd row
    Person(name="D", age=4)  # 4th row
]
```

then the `values` would look like this:
```
"values": [
    ["=ROW()", "A", 1],
    ["=ROW()", "B", 2],
    ["=ROW()", "C", 3],
    ["=ROW()", "D", 4],
]
```

#### Select

Returns the number of rows that matched the given condition.

The operation method should expect these parameters:
| Parameter Name | Required | Remarks                                                                                                                                                     |
| -------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `columns`      | No       | List of columns that we will fetch. Defaults to return all columns if not provided.                                                                         |
| `conditions`   | No       | Condition that must be true for the row to be returned in [Google Sheet Query Language][QueryLanguage] format. Defaults to return all rows if not provided. |
| `limit`        | No       | Impose a limit on the number of rows that will be returned. Defaults to no limit if not provided.                                                           |
| `offset`       | No       | Skips the given number of first rows from being returned. Defaults to not skip any rows if not provided.                                                    |
| `order_by`     | No       | Sorts the row based on the given column values in either ascending or descending order. Defaults to not sort the rows.                                      |

We will utilise the [GViz API][GVizAPI] to return the matching rows from the spreadsheet by running the Google Query below:

`SELECT <column-1>, <column-2>, ..., <column-n> WHERE A IS NOT NULL [AND <condition>] [ORDER BY <ordering-1>, <ordering-2>, ..., <ordering-n>] [LIMIT <limit>] [OFFSET <offset>]`

Note: Query inside square bracket denotes an optional query, only need to be provided if necessary.

- Replace the `<column-1>, <column-2>, ..., <column-n>` based on the columns that are specified in the `columns` parameter.
- If the client provided the `conditions` parameter, replace  `<condition>` with the actual client's condition, e.g.
  suppose the `conditions` is `"B > 5"` then the `WHERE` clause will be equal to `WHERE A IS NOT NULL AND B > 5`.
- If the client provided `limit` or `offset` parameter,  replace `<limit>` and `<offset>` with the given value respectively.
- If the client provided `order_by` parameter, replace the `<ordering-1>, <ordering-2>, ..., <ordering-n>` with the
  value given by the client, e.g. suppose the `order_by` parameter is `A ASC, B DESC` then the `ORDER BY` clause will
  be equal to `ORDER BY A ASC, B DESC`.

On how to use GViz API can refer to the appendix below.

#### Update

Update selected rows with the given value.

The operation method should expect these parameters:
| Parameter Name | Required | Remarks                                                                                                                                                             |
| -------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `value`        | Yes      | A map of column to the updated value that we're going to replace the rows with.                                                                                     |
| `conditions`   | No       | Condition that must be true in order for the row to be updated in [Google Sheet Query Language][QueryLanguage] format. Defaults to update all rows if not provided. |

To get the list of affected rows, we need to call GViz API with the following query:
`SELECT A WHERE A IS NOT NULL [AND <condition>]`

Note: `A` column is referring to the `_rid` column (which contains the row's index).

After we get the list of indices that we need to update, call [spreadsheets.values.batchUpdate][BatchUpdateAPI] to
update all the affected rows with the given `value`. Note that we should only update the columns that is specified in
the `value` (suppose `value` is `{"col1": "1", "col3": "2"}` and in the spreadsheet we have 3 columns `col1`, `col2`
and `col3`, we should not touch the `col2`), this means that you need to create 2 update query to update `col1` and
`col3` for each affected rows when calling the API.

#### Delete

Delete rows that matched the given conditions.

The operation method should expect these parameters:

| Parameter Name | Required | Remarks                                                                                                                                                             |
| -------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `conditions`   | No       | Condition that must be true in order for the row to be deleted in [Google Sheet Query Language][QueryLanguage] format. Defaults to delete all rows if not provided. |


To get the list of affected rows, we need to call GViz API with the following query:
`SELECT A WHERE A IS NOT NULL [AND <conditions>]`

Note: `A` column is referring to the `_rid` column (which contains the row's index).

After we get the list of row indices that we need to delete (by reading the value of `_rid` column returned by previous
query), call [spreadsheets.values.clear][ClearAPI] API to remove all the affected rows.

#### Count

Return the number of rows that matched the given conditions.

The operation method should expect these parameters:

| Parameter Name | Required | Remarks                                                                                                                                                            |
| -------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `conditions`   | No       | Condition that must be true in order for the row to be counted in [Google Sheet Query Language][QueryLanguage] format. Defaults to count all rows if not provided. |

To get the number of rows, we need to call GViz API with the following query:
`SELECT COUNT(A) WHERE A IS NOT NULL [AND <conditions>]`

### Appendix

#### Calling GViz API

To call the GViz API create a `GET` request to `https://docs.google.com/spreadsheets/d/<spreadsheet_id>/gviz/tq`
(replace `<spreadsheet_id>` with the spreadsheet ID that we're going to operate on) with the following query parameters
and headers:

| Parameter Name | Value                     |
| -------------- | ------------------------- |
| `sheet`        | `<sheet_name>`            |
| `tqx`          | `responseHandler:freeleh` |
| `tq`           | `<gsheet_query>`          |
| `headers`      | `1`                       |

| Header Name   | Value                 |
| ------------- | --------------------- |
| Content-Type  | `application/json`    |
| Authorization | `Bearer <auth_token>` |

Replace `<sheet_name>` with the sheet name that we're going to operate on, `<gsheet_query>` with the query that we
want to run, and `<auth_token>` with the token that you use to call the Google Sheet V4 API.

For example, suppose you want to run the "SELECT A" query on `Sheet1` of a spreadsheet with ID `ABCD` with `TOKEN` as
the authorization token the actual request will look like this:
```
GET /spreadsheets/d/ABCD/gviz/tq?sheet=Sheet1&tqx=responseHandler:freeleh&tq=SELECT%20A&headers=1 HTTP/2
Host: docs.google.com
Content-Type: application/json
Authorization: Bearer TOKEN
```

If everything works well, you will get a response like this:
```
/*O_o*/
freeleh({"version":"0.6","reqId":"0","status":"ok","sig":"880399325","table":{"cols":[{"id":"B","label":"name","type":"string"},{"id":"C","label":"age","type":"number","pattern":"General"},{"id":"D","label":"date of birth","type":"date","pattern":"m-d-yyyy"}],"rows":[{"c":[{"v":"name1"},{"v":10.0,"f":"10"},{"v":"Date(1999,0,1)","f":"1-1-1999"}]},{"c":[{"v":"name2"},{"v":11.0,"f":"11"},{"v":"Date(2000,0,1)","f":"1-1-2000"}]},{"c":[{"v":"name3"},{"v":12.0,"f":"12"},{"v":"Date(2001,0,1)","f":"1-1-2001"}]}],"parsedNumHeaders":0}});
```

Before we can process the response, we need to discard any character that comes before the first `{` character and
those that come after the last `}` character, after we discard those characters we will get a valid JSON string.

```
{"version":"0.6","reqId":"0","status":"ok","sig":"880399325","table":{"cols":[{"id":"B","label":"name","type":"string"},{"id":"C","label":"age","type":"number","pattern":"General"},{"id":"D","label":"date of birth","type":"date","pattern":"m-d-yyyy"}],"rows":[{"c":[{"v":"name1"},{"v":10.0,"f":"10"},{"v":"Date(1999,0,1)","f":"1-1-1999"}]},{"c":[{"v":"name2"},{"v":11.0,"f":"11"},{"v":"Date(2000,0,1)","f":"1-1-2000"}]},{"c":[{"v":"name3"},{"v":12.0,"f":"12"},{"v":"Date(2001,0,1)","f":"1-1-2001"}]}],"parsedNumHeaders":0}}
```

`.table.cols` contain the list of column info objects that we included in the `SELECT` clause in the same order
of appearance.

`.table.rows` contain the list of row objects. The value of a `i`-th row can be accessed via `.table.rows[i].c` key.
Inside of the row value object you will find `v` and `f` key, where `v` corresponds to the raw value while `f`
corresponds to the formatted value (the value that user can see from the Google Sheet GUI).

To parse the value, use the `v` value of each cell and convert the value to your language native type.

| Data type | `v` Value Example                 | Recommendation                                                                                     |
| --------- | --------------------------------- | -------------------------------------------------------------------------------------------------- |
| Bool      | `"true"` or `"false"`             | Treat `"true"` as truthy boolean value and `"false"` as falsy.                                     |
| Number    | `"1234"`, `"1.4"`, `"1.234573E8"` | Parse them as double-precision floating point value (and then convert to other type if necessary). |
| String    | `"value"`                         | Parse them as string.                                                                              |

## Key-Value (KV) Store

KV Store is a data structure that can map a key to a value (similar to a hash map).

We will build KV Store on top of our Row Store. The row data schema for each record looks like this (in `PyFreeDB` model):
```
class Entry(models.Model):
    key = models.StringField()
    value = models.StringField()
```
### Operations

#### Set

Set the value of a `key` to the given `value`.

The operation method should expect these parameters:

| Parameter Name | Required | Remarks                       |
| -------------- | -------- | ----------------------------- |
| `key`          | Yes      | Record's key                  |
| `value`        | Yes      | The value that we want to set |


For both Default and Append Only mode, simply insert `Entry(key=<key>, value=<value>)` to the row store.

#### Get

Get the value of a `key` from the store.

The operation method should expect these parameters:

| Parameter Name | Required | Remarks                          |
| -------------- | -------- | -------------------------------- |
| `key`          | Yes      | Record's key that we want to get |

1. For Default Mode, `Get` can be done by selecting rows that matched `key=<key>` condition and applying `limit=1`.
	- In `PyFreeDB` syntax, it's equivalent to: `row_store.select().where("key = ?", <key>).limit(1).execute()`.

2. For Append-Only mode, `Get` can be done by selecting rows that matched `key=<key>` sorted by `_rid` value in
   descending manner and applying `limit=1`. Note that if we get `value=""` we must treat it as `KeyNotFound`.
	- In `PyFreeDB` syntax, it's equivalent to: `row_store.select().where("key = ?", key).order_by(Ordering.DESC("_rid")).limit(1).execute()`.


#### Delete

Deletes the record associated with the given `key` from the store.

The operation method should expect these parameters:
| Parameter Name | Required | Remarks                             |
| -------------- | -------- | ----------------------------------- |
| `key`          | Yes      | Record's key that we want to delete |

1. For default mode, `Delete` can be done by deleting row that matched with `key=<key>`.
	- In `PyFreeDB` syntax, it's equivalent to `row_store.delete().where("key = ?", key).execute()`.
2. For Append-Only mode, `Delete` can be done by inserting `Entry(key=<key>, value="")` to the store.
	- In `PyFreeDB` syntax, it's equivalent to `row_store.insert(Entry(key=key, value="")).execute()`


[AppendAPI]: https://developers.google.com/sheets/api/reference/rest/v4/spreadsheets.values/append
[BatchUpdateAPI]: https://developers.google.com/sheets/api/reference/rest/v4/spreadsheets.values/batchUpdate
[ClearAPI]: https://developers.google.com/sheets/api/reference/rest/v4/spreadsheets.values/clear

[GVizAPI]: https://developers.google.com/chart/interactive/docs/reference
[A1Notation]: https://developers.google.com/sheets/api/guides/concepts
[QueryLanguage]: https://developers.google.com/chart/interactive/docs/querylanguage
