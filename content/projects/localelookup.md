---
title: "LocaleLookup"
tagline: "Yelp, but much smaller"
category: "Web Dev"
status: "archived"
type: "web-app"
description: "A PERN-stack CRUD app for looking up local businesses. Built to learn the stack end to end."
language: "JavaScript"
tech:
  - "PostgreSQL"
  - "Express.js"
  - "React"
  - "Node.js"
github: "https://github.com/AbuCTF/LocaleLookup"
draft: false
---

An information retrieval site for local businesses, loosely shaped like Yelp. Built over February 2024 to learn PERN end to end.

## What it is

One table, five routes, a React front end. Locations have a name, an address, and a price range from 1 to 5 that renders as `$` through `$$$$$`.

```sql
CREATE TABLE locations (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    address VARCHAR(50) NOT NULL,
    priceRange INT NOT NULL check(priceRange >= 1 and priceRange <= 5)
);
```

The API is the full CRUD set under `/api/v1/location`. Every query is parameterised (`$1`, `$2`) rather than concatenated, and inserts and updates both use `returning *` so the client gets the new row back without a second round trip.

## Why PostgreSQL

I picked Postgres over Mongo deliberately. The data is relational and fixed-shape, the price range wants a `CHECK` constraint, and the moment you add reviews and categories you want joins rather than embedded documents.

The connection pool takes no config at all:

```javascript
const { Pool } = require("pg");
const pool = new Pool({});

module.exports = {
    query: (text, params) => pool.query(text, params),
};
```

An empty `Pool({})` reads `PGHOST`, `PGUSER`, `PGDATABASE`, `PGPASSWORD` and `PGPORT` from the environment. Nothing to wire up.

## Where it stopped

It stopped at scaffolding. The front end has a header, an add-location form and a placeholder for the list. Map integration with Google Maps or Mapbox was the plan and never happened.

Every route also swallows its errors:

```javascript
} catch (err) {
    console.log(err);
}
```

Nothing is sent back, so a failed query leaves the request hanging until it times out. It's the first thing I'd fix.

There's a longer writeup on [Notion](https://deadgawk.notion.site/LocaleLookup-Information-Retrieval-Website-4b0a80f4b4424339a0dbd31c8d439911).
