MSSQL JSON support examples for mod_booking

This document contains example SQL snippets for SQL Server (MSSQL / sqlsrv) that correspond to changes added in the codebase.

1) Extract scalar from JSON object (JSON_VALUE)

Example: extract `price` from a JSON object stored in column `json`:

SELECT JSON_VALUE(json, '$.price') AS price

Cast to numeric:
SELECT CAST(JSON_VALUE(json, '$.price') AS FLOAT) AS price

2) Extract key from JSON array element at index (JSON_VALUE)

Example: extract `key` from first element of JSON array in column `json`:

SELECT JSON_VALUE(json, '$[0].key') AS first_key

3) Expand payments array and extract fields (OPENJSON + JSON_VALUE)

Assume `sch.json` contains {"installments": {"payments": [ {...}, {...} ] }}

SELECT
  bo.id AS optionid,
  sch.userid,
  CAST(JSON_VALUE(pay.[value], '$.timestamp') AS INT) AS datefield,
  CAST(JSON_VALUE(pay.[value], '$.paid') AS FLOAT) AS paid,
  CAST(JSON_VALUE(pay.[value], '$.price') AS FLOAT) AS price,
  CAST(JSON_VALUE(pay.[value], '$.id') AS INT) AS payment_id
FROM local_shopping_cart_history sch
CROSS APPLY OPENJSON(sch.json, '$.installments.payments') AS pay
WHERE CAST(JSON_VALUE(pay.[value], '$.paid') AS FLOAT) = 0

4) Check membership in JSON array (OPENJSON)

Assume `json` contains { "sharedplaceswithoptions": [ 1, 2, 3 ] }

SELECT id FROM booking_options
WHERE EXISTS (
  SELECT 1 FROM OPENJSON(json, '$.sharedplaceswithoptions') AS sp
  WHERE sp.[value] = '42' -- checks membership of option id 42
)

5) Aggregate JSON objects per user+option (FOR JSON PATH)

Create an aggregated JSON array of certificate objects per user + optionid:

SELECT t.userid,
       CAST(JSON_VALUE(t.data, '$.bookingoptionid') AS INT) AS optionid,
       (
         SELECT t2.id, t2.code, t2.expires, t2.data, t2.timecreated
         FROM tool_certificate_issues t2
         WHERE t2.userid = t.userid
           AND CAST(JSON_VALUE(t2.data, '$.bookingoptionid') AS INT) = CAST(JSON_VALUE(t.data, '$.bookingoptionid') AS INT)
         FOR JSON PATH
       ) AS certificate
FROM tool_certificate_issues t
GROUP BY t.userid, CAST(JSON_VALUE(t.data, '$.bookingoptionid') AS INT)

Notes
- SQL Server functions used: JSON_VALUE, JSON_QUERY, OPENJSON, FOR JSON PATH.
- OPENJSON returns rows with columns `key`, `value`, `type`. For arrays of scalars, `value` contains the scalar value.
- JSON_VALUE returns NVARCHAR; cast when numeric types are required.

Testing tips
- Use `SELECT JSON_QUERY('[{"id":1,"price":10}]', '$[0]')` to inspect JSON fragments.
- Validate that the stored JSON is valid nvarchar (not stored as binary) and accessible by JSON functions.

