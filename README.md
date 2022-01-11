# Firebolt Node.js SDK

<img width="1114" alt="Screen Shot 2022-01-10 at 10 56 30 AM" src="https://user-images.githubusercontent.com/7674553/148823093-5ff07a44-9954-4640-b322-a7e029e47450.png">

## Installation

This library is published in the NPM registry and can be installed using any compatible package manager.

```sh
npm install firebolt-sdk --save

# For Yarn, use the command below.
yarn add firebolt-sdk
```


## Using the library

```typescript
import Firebolt from 'firebolt-sdk'

const firebolt = Firebolt();

const connection = await firebolt.connect({
  username: process.env.FIREBOLT_USERNAME,
  password: process.env.FIREBOLT_PASSWORD,
  database: process.env.FIREBOLT_DATABASE,
  engineName: process.env.FIREBOLT_ENGINE_NAME
});

const statement = await connection.execute("SELECT 1");

// fetch statement result
const { data, meta } = await statement.fetchResult();

// or stream result
const { data } = await statement.streamResult();

data.on("metadata", metadata => {
  console.log(metadata);
});

data.on("error", error => {
  console.log(error);
});

const rows = []

for await (const row of data) {
  rows.push(row);
}

console.log(rows)


```

## Contents
* [About](#about)
* [Documentation](#documentation)
* [Usage](#usage)
  * [Firebolt client](#usage-firebolt-client)
  * [Create connection](#usage-create-connection)
    * [ConnectionOptions](#usage-create-connection-options)
  * [Test connection](#usage-test-connection)
  * [Engine URL](#usage-engine-url)
  * [Execute query](#execute-query)
    * [ExecuteQueryOptions](#execute-query-options)
    * [QuerySettings](#execute-query-settings)
    * [ResponseSettings](#execute-query-response)
  * [Fetch result](#fetch-result)
  * [Stream result](#stream-result)
  * [Result hydration](#result-hydration)
  * [Engine management](#engine-management)
    * [getById](#engine-get-by-id)
    * [getByName](#engine-get-by-name)
    * [Engine](#engine)
      * [start](#engine-management-start)
      * [stop](#engine-management-stop)
      * [restart](#engine-management-restart)
* [Recipes](#recipes)
  * [Streaming results](#recipes-stream)
  * [Custom stream transformers](#recipes-stream-transform)


<a name="About"></a>
## About
The Firebolt client for Node.js. firebolt-sdk provides common methods for quering Firebolt databases, fetching and streaming results, and engine management.

firebolt-sdk supports  Node.js `> v14`.

<a name="documentation"></a>
## Documentation

<a name="usage"></a>
## Usage

<a name="usage-create-connection"></a>
### Create connection

```typescript
const connection = await firebolt.connect(connectionOptions);
```

<a name="usage-create-connection-option"></a>
#### ConnectionOptions
```typescript
type ConnectionOptions = {
  username: string;
  password: string;
  database: string;
  engineName?: string;
  engineEndpoint?: string;
};
```

<a name="usage-create-connection"></a>
### Test connection
TODO: write motivation
connection can be tested using:
```typescript
const firebolt = Firebolt();
await firebolt.testConnection(connectionOptions)
```
which will perform authentication and simple `select 1` query

<a name="usage-engine-url"></a>
### Engine URL
Firebolt engine URLs use the following format:

```
<engine-name>.<account-name>.<region>.app.firebolt.io
```

For example: `your-engine.your-account.us-east-1.app.firebolt.io`. You can find and copy your engine endpoint name in the Firebolt web UI.

<a name="execute-query"></a>
### Execute Query

```typescript
const statement = await connection.execute(query, executeQueryOptions);
```

<a name="execute-query-options"></a>
### ExecuteQueryOptions

```typescript
export type ExecuteQueryOptions = {
  settings?: QuerySettings;
  response?: ResponseSettings;
};
```

<a name="execute-query-settings"></a>
### QuerySettings

| Parameter     | Required | Default      | Description                       |
|---------------|----------|--------------|-----------------------------------|
| output_format |          | JSON_COMPACT | Specifies format of selected data |


<a name="execute-query-response"></a>
### ResponseSettings

| Parameter         | Required | Default | Description                 |
|-------------------|----------|---------|-----------------------------|
| normalizeData     |          | false   |                             |
| bigNumberAsString |          | false   | hydrate BigNumber as String |


<a name="fetch-result"></a>
### Fetch result

```typescript
const { data, meta, statistics } = await statement.fetchResult();
```

The Promise API **is not recommended** for `SELECT` queries with large result sets (greater than 10,000 rows). This is because it parses results synchronously, so will block the JS thread/event loop and may lead to memory leaks due to peak GC loads.

It is recommended to use `LIMIT` in your queries when using the Promise API.

<a name="stream-result"></a>
### Stream result

```typescript
const { data } = await statement.streamResult();
const rows: unknown[] = [];

data.on("metadata", metadata => {
  console.log(metadata);
});

data.on("error", error => {
  console.log(error);
});

for await (const row of data) {
  rows.push(row);
}
```

<a name="result-hydration"></a>
### Result hydration

firebolt-sdk maps SQL data types to their corresponding JavaScript equivalents. The mapping is described in the table below:

| Category    | SQL type | JavaScript type | Notes                                                                                                                             |
|-------------|----------|-----------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Numeric     | INT      | Number          | If value cannot be represented by JavaScript Number (determine using Number.isSafeInteger), BigNumber from "bignumber.js" is used |
|             | INTEGER  | Number          |                                                                                                                                   |
|             | BIGINT   | Number          |                                                                                                                                   |
|             | LONG     | Number          |                                                                                                                                   |
|             | FLOAT    | Number          |                                                                                                                                   |
|             | DOUBLE   | Number          |                                                                                                                                   |
| String      | VARCHAR  | String          |                                                                                                                                   |
|             | TEXT     | String          |                                                                                                                                   |
|             | STRING   | String          |                                                                                                                                   |
| Date & Time | DATE     | Date            |                                                                                                                                   |


<a name="engine-management"></a>
### Engine management

Engines can be managed by using the `resourceManager` object.

```typescript
const firebolt = Firebolt();
const enginesService = firebolt.resourceManager.engines
```

<a name="engine-get-by-id"></a>
#### getById

Returns engine using engine ID and account ID.

```typescript
const firebolt = Firebolt();
const engine = await firebolt.resourceManager.engines.getById(
  "c8a228ea-93df-4784-99f9-a99368518782",
  "a32b073b-e093-4880-8fd4-3b302b4cf221"
);
```

<a name="engine-get-by-name"></a>
#### getByName

Returns engine using engine name.

```typescript
const firebolt = Firebolt();
const engine = await firebolt.resourceManager.engines.getByName("engine_name")
```

<a name="engine"></a>
#### Engine

| Property                 | Type                                      | Notes |
|--------------------------|-------------------------------------------|-------|
| `id`                     | `{engine_id: string; account_id: string}` |       |
| `name`                   | `string`                                  |       |
| `endpoint`               | `string`                                  |       |
| `current_status_summary` | `string`                                  |       |
|                          |                                           |       |

<a name="engine-management-start"></a>
##### Start

Starts an engine.

```typescript
const firebolt = Firebolt();
await firebolt.connect(connectionOptions);
const engine = await firebolt.resourceManager.engines.getByName("engine_name")
await engine.start()
```

<a name="engine-management-stop"></a>
##### Stop

Stops an engine.

```typescript
const firebolt = Firebolt();
await firebolt.connect(connectionOptions);
const engine = await firebolt.resourceManager.engines.getByName("engine_name")
await engine.stop()
```

<a name="engine-management-restart"></a>
##### Restart

Restarts an engine.

```typescript
const firebolt = Firebolt();
await firebolt.connect(connectionOptions);
const engine = await firebolt.resourceManager.engines.getByName("engine_name")
await engine.restart()
```

<a name="recipes"></a>
## Recipes

<a name="recipes-stream"></a>
### Streaming results

The recommended way to consume query results is by using streams.

For convenience, `statement.streamResult` also returns `meta: Promise<Meta[]>` and `statistics: Promise<Statistics>`, which are wrappers over `data.on('metadata')` and `data.on('statistics')`.

```typescript
const firebolt = Firebolt();

const connection = await firebolt.connect(connectionParams);

const statement = await connection.execute("SELECT 1");

const {
  data,
  meta: metaPromise,
  statistics: statisticsPromise
} = await statement.streamResult();

const rows: unknown[] = [];

const meta = await metaPromise;

for await (const row of data) {
  rows.push(row);
}

const statistics = await statisticsPromise

console.log(meta);
console.log(statistics);
console.log(rows)
```

<a name="recipes-stream-transformers"></a>
### Custom stream transformers

To achieve seamless stream pipes to `fs` or `stdout`, you can use the `Transform` stream.

```typescript
import stream,  { TransformCallback } from 'stream';

class SerializeRowStream extends stream.Transform {
  public constructor() {
    super({
      objectMode: true,
      transform(
        row: any,
        encoding: BufferEncoding,
        callback: TransformCallback
      ) {
        const transformed = JSON.stringify(row);
        this.push(transformed);
        this.push('\n')
        callback();
      }
    });
  }
}

const serializedStream = new SerializeRowStream()

const firebolt = Firebolt();
const connection = await firebolt.connect(connectionParams);
const statement = await connection.execute("select 1 union all select 2");

const { data } = await statement.streamResult();

serializedStream.pipe(serializedStream).pipe(process.stdout);
```

Or use `rowParser` that returns strings or Buffer:

```typescript
const { data } = await statement.streamResult({
  rowParser: (row: string) => `${row}\n`
});

data.pipe(process.stdout);
```

## Development process
### Actions before
### Setup env variables

  ```sh
  cp .env.example .env
  ```
  
## Execute tests

  ```sh
    npm test
  ```

## License

Released under [Apache License](./LICENSE).
