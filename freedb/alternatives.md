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
2. For [retrieval](https://github.com/SteinHQ/Stein/blob/master/controllers/readSheet.js), [update](https://github.com/SteinHQ/Stein/blob/master/controllers/editRow.js), and [deletion](https://github.com/SteinHQ/Stein/blob/master/controllers/deleteRow.js), the following is how it works:
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

## [`sheetsql`](https://github.com/joway/sheetsql)

## [`gooss`](https://github.com/Stuk/gooss)

## [`drive-db`](https://github.com/franciscop/drive-db)