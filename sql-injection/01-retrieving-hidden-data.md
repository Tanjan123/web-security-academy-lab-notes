# SQL Injection in WHERE Clause – Retrieval of Hidden Data

## Lab
PortSwigger Web Security Academy  
Topic: SQL Injection

## What?

A SQL injection vulnerability exists in the product category filter.
User-controlled input is incorporated into a SQL WHERE clause, allowing
an attacker to alter the intended database query.

## Where?

- Endpoint: `/filter`
- Method: `GET`
- Parameter: `category`
- Authentication: Not required

## How did I identify it?

I intercepted the category-selection request in Burp Suite and modified
the `category` parameter.

Adding a single quote (`'`) caused the application to return an HTTP
500 Internal Server Error.

This indicated that my input might be affecting the syntax of a
server-side SQL statement.

## How did I verify it?

I tested the `category` parameter with:

[put your payload here]

The application then returned products that were normally hidden,
confirming that the application's query logic could be manipulated
through the parameter.

## Why does it work?

The application incorporates user-controlled input into a SQL query
without safely separating the input from SQL syntax.

Conceptually, the application performs a query similar to:

[put the conceptual SQL query here]

Because the input is interpreted as part of the SQL statement, an
attacker can modify the logic of the WHERE clause.

## Impact

In this lab, the vulnerability allows an unauthenticated user to bypass
the intended product filter and retrieve unreleased products.

The impact demonstrated by this lab is unauthorized access to data that
the application intended to hide.

## Conditions / Limitations

- No authentication is required.
- The injection occurs in a string context.
- The tested parameter is `category`.
- Further database access was not established as part of this lab.

## Remediation

Use parameterized queries/prepared statements so that user-controlled
values are treated as data rather than executable SQL syntax.

Additional defense-in-depth measures include:

- Applying least privilege to the database account.
- Avoiding verbose database errors in production.
- Validating input according to the application's expected values.

## What I Learned

An HTTP 500 response after introducing a SQL metacharacter can be a
useful clue, but the error alone does not prove SQL injection.

The stronger evidence came from deliberately modifying the query's
logical behavior and observing normally hidden records being returned.
