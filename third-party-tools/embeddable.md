---
title: Embeddable
description:
  Embeddable is a developer toolkit for building fast, interactive, fully-custom
  analytics experiences directly into your app.
---

Embeddable is a developer toolkit for building fast, interactive customer-facing
analytics. It works well with a high performance time-series database like
QuestDB.

In [Embeddable](https://embeddable.com/) define
[Data Models](https://trevorio.notion.site/Data-modeling-35637bbbc01046a1bc47715456bfa1d8)
and
[Components](https://trevorio.notion.site/Using-components-761f52ac2d0743b488371088a1024e49)
in code stored in your own code repository, then use the **SDK** to make these
available for your team in the powerful Embeddable **no-code builder.** The end
result is the ability to deliver fast, interactive **customer-facing analytics**
directly into your product.

Built-in **row-level security** means that every user only ever sees **exactly**
the data they’re allowed to see. And two levels of fully-configurable
**caching** mean you can deliver fast, realtime analytics at scale.

## Prerequisites

- A running QuestDB instance
  - Not running yet? See the [quick start](/docs/quick-start/)

## Getting started with Embeddable

Add a database connection using Embeddable API. This connection connects to your
QuestDB instance. To add a connection, use the following API call:

```javascript
// for security reasons, this must *never* be called from your client-side
fetch("https://api.embeddable.com/api/v1/connections", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Accept: "application/json",
    Authorization: `Bearer ${apiKey}` /* keep your API Key secure */,
  },
  body: JSON.stringify({
    name: "my-questdb-db",
    type: "questdb",
    credentials: {
      host: "my.questdb.host",
      port: "8812",
      user: "admin",
      password: "quest",
    },
  }),
})
```

In response you will receive:

```json
Status 201 { errorMessage: null }
```

The above represents a `CREATE` action, but all `CRUD` operations are available.

The `apiKey` can be found by clicking “**Publish**” on one of your Embeddable
dashboards.

The `name` is a unique name to identify this **connection**.

- By default your **data models** will look for a **connection** called
  “default”, but you can supply models with different
  [**data_source**](https://cube.dev/docs/reference/data-model/cube#data_source)
  names to support connecting different **data models** to different
  **connections**. To do so , specify the
  **[data_source](https://cube.dev/docs/reference/data-model/cube#data_source)**
  name in the model.

The `type` tells Embeddable which driver to use, in this case `questdb`. You can
also connect multiple datasources like `postgres`, `bigquery` or `mongodb`. For
a full list, see
[the documentaiton](https://cube.dev/docs/product/configuration/data-sources).

The `credentials` is a javascript object containing the credentials expected by
the driver:

- Credentials are securely encrypted and only used to retrieve exactly the data
  described in the data models.
- Emeddable strongly encourages you to create a **read-only** database user for
  each connection. Embeddable will only ever read from your database, not write.

To support connecting to different databases for prod, qa, test, etc, or to
support different databases for different customers, you can assign each
**connection** to an **environment**. For more information, see
[Environments API](https://www.notion.so/Environments-API-497169036b5148b38f7936aa75e62949?pvs=21).
