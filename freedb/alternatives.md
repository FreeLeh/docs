# Alternatives

There are a few notable other alternatives that we believe are aiming to do something similar as `FreeDB`.
We would like to try our best to summarise how these other alternatives work and how it compares with `FreeDB`.

> If you find there is anything inaccurate about the analysis below,
> please feel free to let us know and we can discuss it further.

## Summary

Here is the `tl;dr` version. Please read further for more details.

TODO(edocsss): need to mention that all implementations are good in terms of readability + leveraging Google Sheets capabilities.

## [`Stein`](https://github.com/SteinHQ/Stein)

`Stein` has a backend server as a proxy.

1. It provides both hosted and self-managed backend option.
2. Hosted backend offers higher capacity (more requests per month and more rows per sheet) at cost.
3. The backend server provides a few REST API that can be called directly or using the official JavaScript library.

This project provides the following operations.

1. Retrieve rows with conditions, limit, and offset.
2. Insert new rows.
3. Update rows with conditions and limit.
4. Delete rows with conditions and limit.

A few important implementation details to take note.

1. The conditions only support [equality check](https://github.com/SteinHQ/Stein/blob/master/controllers/objectDoesMatch.js) (using JavaScript `===` check).
2. For [retrieval](https://github.com/SteinHQ/Stein/blob/master/controllers/readSheet.js),
   [update](https://github.com/SteinHQ/Stein/blob/master/controllers/editRow.js), and
   [deletion](https://github.com/SteinHQ/Stein/blob/master/controllers/deleteRow.js), the following is how it works:
   - [Retrieve all rows](https://github.com/SteinHQ/Stein/blob/master/controllers/retrieveSheet.js) from Google Sheet.
   - Perform the condition, offset, and limit operation in-memory.
   - Either return the matching rows or update/delete the matching rows.
3. The server has a dependency on a MongoDB instance to store client information.

### 

## [`Serve`](https://www.withserve.com/)

`Serve` has a backend server as a proxy.

1. It only provides hosted backend option (currently it seems it is provided for free as of 24 August 2022).
2. It provides integrations with many data storages, not just Google Sheets (e.g. Airtable, MySQL, PostgreSQL, etc.).
3. There is no official client library provided, all REST APIs provided must be called manually.

> Although `Serve` provides integrations with many different data storages, we are going to just
> focus on the Google Sheet capabilities.

This project provides the following operations for Google Sheet.

1. Retrieve all rows.
2. Insert new rows.
3. Update rows.
4. Delete rows.

A few important implementation details to take note.

1. Retrieving rows must include all rows. It does not support filtering, offsetting, or limiting.
2. No batch row insertion support as of 24 August 2022.
3. A row can only be updated as a full row (cannot update only selected columns).
4. If we want to update or delete a row, we must provide the row index.

## [`SheetDB`](https://sheetdb.io/)

`SheetDB` has a backend server as a proxy.

1. It only provides hosted backend option. 
2. It provides both free and paid tier.

This project provides the following operations.

1. Retrieve rows with conditions, limit, offset, and order by a specific column.
2. Retrieve keys (similar to column names).
3. Count the number of rows.
4. Insert new rows.
5. Update rows with conditions and limit.
6. Delete rows with conditions and limit.
7. Delete duplicated rows within a sheet (the row must have the same exact content).
8. Retrieve specific cell values.
9. Insert new sheets.
10. Delete sheets.

A few important implementation details to take note.

1. Batch update is only supported in the paid version.
2. The project defines their own [condition format](https://docs.sheetdb.io/#get-search-in-document) for retrieval, deletion ,and update.
3. The project supports `handlebars` to display data directly in HTML.
4. The project supports [data caching](https://docs.sheetdb.io/#caching) inside their hosted server (paid version only).
   - This means when the data is cached, changes in Google Sheet (directly changed) is not going to be reflected immediately.
   - However, if the data is updated/inserted/deleted via the APIs provided by `SheetDB`, the cache will be invalidated.

## [`sheetsql`](https://github.com/joway/sheetsql)

`sheetsql` is a TypeScript library that treats Google Sheets as a database. There is no backend server as a proxy.

This project provides the following operations.

1. Retrieve rows with conditions.
2. Insert new rows.
3. Update rows with conditions.
4. Delete rows with conditions.

A few important implementation details to take note.

1. The conditions only support [string equality check](https://github.com/joway/sheetsql/blob/master/src/storage.ts#L77).
2. For [retrieval](https://github.com/joway/sheetsql/blob/master/src/storage.ts#L108),
   [update](https://github.com/joway/sheetsql/blob/master/src/storage.ts#L134), and
   [delete](https://github.com/joway/sheetsql/blob/master/src/storage.ts#L173), the following is how it works:
   - [Retrieve all rows](https://github.com/joway/sheetsql/blob/master/src/storage.ts#L209) from Google Sheet (data is cached in-memory).
   - Perform the [condition operation](https://github.com/joway/sheetsql/blob/master/src/storage.ts#L61) in-memory.
   - Either return the matching rows or update/delete the matching rows.
3. The [update](https://github.com/joway/sheetsql/blob/master/src/storage.ts#L148) and [delete](https://github.com/joway/sheetsql/blob/master/src/storage.ts#L180)
   operations are done by calling the relevant Google Sheets API one-by-one for each row.
   - Google Sheets API actually supports batch update and delete.
4. The project supports [data caching](https://github.com/joway/sheetsql/blob/master/src/storage.ts#L201) via in-memory caching.
   - This means when the data is cached, changes in Google Sheet (directly changed) is not going to be reflected immediately.
   - For write based operations (insert, update, and delete), both local and remote data will be updated.

## [`gooss`](https://github.com/Stuk/gooss)

`gooss` is a JavaScript library that reads data from Google Sheets.

The project only supports reading all rows and applying a client provided callback for each row.

A few important implementation details to take note.

1. It only supports full data retrieval.
2. It does not support condition matching.
3. It supports HTML templating using [`Underscore.js`](https://underscorejs.org/#template).


## [`drive-db`](https://github.com/franciscop/drive-db)

`drive-db` is a JavaScript library that reads data from Google Sheets.

The project only supports reading all rows.

A few important implementation details to take note.

1. It only supports full data retrieval.
2. It does not support condition matching.
3. The project supports [data caching](https://github.com/franciscop/drive-db/blob/master/index.js#L49) via in-memory caching.
   - This means when the data is cached, changes in Google Sheet (directly changed) is not going to be reflected immediately.

> Note that this project is already deprecated as it depends on Google Sheets v3 API.