# SQL/Database

Always use table and column names in English, plural, and in snake_case
For primary key use UUID type
For primary and foreign keys, always use the table name in the singular followed by _id, for example: users -> user_id, customers -> customer_id, orders -> order_id, payments -> payment_id
Use uppercase for reserved words, for example, SELECT, FROM, JOIN, WHERE
Always use join instead of selecting the tables and performing the join in the where clause
If possible, perform the join using on instead of on
Never use * in the select statement; always make it clear which columns are being returned
For string types, always use text, do not use varchar
For numeric types, use int or numeric depending on whether it is a floating-point type
For dates, use the timestamptz type
When a column is used as a search, create an index
Whenever If possible, resolve grouping or ordering issues in the query itself with group by and order by.
If using order by, always indicate whether the order is desc or asc.
Always use prepared statements and do not interpolate strings in queries.
Whenever it makes sense, use in and between instead of combinations like and and or.
Break lines after SELECT, FROM, WHERE, GROUP BY, ORDER BY.
Use constraints like NOT NULL whenever it makes sense, aligned with what is being done in the application.
Every table should have created_at and updated_at values.
Whenever you make any changes to the database, create a migration to apply and another to undo if necessary.
use uuidv7 to generate uuids for primary keys