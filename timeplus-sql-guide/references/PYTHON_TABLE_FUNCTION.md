# Python Table Functions in Timeplus

## Overview

Python Table Functions let you embed Python code directly into Timeplus streaming SQL pipelines. They are defined as **external streams** with Python logic inside a `$$ ... $$` block, and support three operation modes: **read** (custom source), **write** (custom sink), and **transform** (stateful or stateless row/batch processing).

Use this reference when the user asks to:
- Connect to a data source or sink that has no native Timeplus connector
- Apply custom Python logic, ML inference, or enrichment to a stream
- Read from a WebSocket, REST API, proprietary SDK, or file format
- Reuse existing Python data-processing code inside SQL

---

## Prerequisites

### Install Python packages using Timeplus SQL (if needed)

Run following SQL to install Python packages:

```sql
SYSTEM INSTALL PYTHON PACKAGE 'package-name';

-- Example:
SYSTEM INSTALL PYTHON PACKAGE 'kafka-python';
SYSTEM INSTALL PYTHON PACKAGE 'websockets';
```

---

## Syntax Template

```sql
CREATE EXTERNAL STREAM <stream_name> (
    <col1> <type1>,
    <col2> <type2>
    -- add more columns as needed
)
AS
$$
import ...

def <read_fn>():
    # yield rows as a tuple for multiple columns
    config_parameter = default_value
    client = setup_client()
    try:
        for item in client.stream():
            yield (col1_value, col2_value, ...) # return tuple for multiple columns
    finally:
        client.close()

def <write_fn>(col1, col2):
    # called with one list per column (vectorized)
    config_parameter = default_value
    client = setup_client()
    try:
        for val1, val2 in zip(col1, col2):
            client.send(val1,val2)
        client.flush()
    finally:
        client.close()

def <init_fn>(params):
    # optional: params is a string, usually JSON from init_function_parameters
    pass

def <flush_fn>():
    # optional for sinks: called on checkpoints and before cleanup
    pass

def <deinit_fn>():
    # optional: called once after read/write processing
    pass

$$
SETTINGS
    type = 'python',
    read_function_name  = '<read_fn>',    -- optional: for source streams
    write_function_name = '<write_fn>',   -- optional: for sink streams
    init_function_name = '<init_fn>',     -- optional
    named_collection = '<collection_name>', -- optional credential/config source
    -- or use direct parameters instead of named_collection:
    -- init_function_parameters = '<string>',
    flush_function_name = '<flush_fn>',   -- optional sink hook
    deinit_function_name = '<deinit_fn>', -- optional cleanup hook
    mode = 'auto'                         -- optional: auto, streaming, batch
;
```

**Key rules:**
- `type = 'python'` is mandatory.
- Python external-stream read/write/transform support is available in Timeplus
  Enterprise 3.2.1+.
- `read_function_name` defaults to the stream name if omitted.
- `write_function_name` defaults to `read_function_name` if omitted.
- All Python code goes inside the `$$ ... $$` heredoc.
- In Timeplus Enterprise 3.2.2+, incoming SQL `string` values are Python
  `bytes` for binary safety. This affects write functions and `python_table()`
  transforms, including nested `array(string)` elements. Decode before text
  processing.
- `init_function_parameters` requires `init_function_name`.
- Prefer `named_collection` for credentials. Use direct `init_function_parameters`
  only for non-secret values or when intentionally overriding an overridable
  collection key. Direct parameters are stored in stream metadata; named
  collections keep credentials out of `SHOW CREATE EXTERNAL STREAM`.
- `flush_function_name` is sink-only. It is called on checkpoints and before
  cleanup. `deinit_function_name` is called once after processing.
- Creating or attaching an external stream that references a named collection
  requires `GRANT NAMED COLLECTION ON *.* TO user_name`.

---

## Credential Pattern — Named Collection + Init Hook

Use a named collection when Python code needs credentials. Put one string value
under the `init_function_parameters` key, usually JSON. The DBMS merges the named
collection into external-stream settings before constructing the Python storage,
then calls the init hook with that string before `read_function_name` or
`write_function_name` runs. This pattern requires Timeplus Enterprise 3.2.1+.

```sql
CREATE NAMED COLLECTION api_source_creds AS
    init_function_parameters = '{"api_key":"secret-token","endpoint":"https://api.example.com/events"}'
    NOT OVERRIDABLE;

CREATE EXTERNAL STREAM api_events (
    event_id string,
    payload  string
)
AS
$$
import json
import requests

API_KEY = ''
ENDPOINT = ''

def _tp_init(params):
    global API_KEY, ENDPOINT
    cfg = json.loads(params)
    API_KEY = cfg['api_key']
    ENDPOINT = cfg['endpoint']

def read_events():
    resp = requests.get(
        ENDPOINT,
        headers={'Authorization': 'Bearer ' + API_KEY},
        timeout=10,
    )
    resp.raise_for_status()
    for item in resp.json():
        yield (str(item.get('id', '')), json.dumps(item))
$$
SETTINGS
    type = 'python',
    read_function_name = 'read_events',
    init_function_name = '_tp_init',
    named_collection = 'api_source_creds';
```

Operational notes:

- `SHOW CREATE EXTERNAL STREAM` keeps `named_collection = 'api_source_creds'`
  rather than expanding the credential string.
- `system.named_collections` can still expose collection values. Restrict named
  collection privileges to operators.
- If the collection lacks `init_function_parameters`, the init hook is called
  with no arguments.
- Named collection values are merged into the stream settings when the external
  stream storage is constructed. Recreate the stream when rotating credentials
  for a long-running Python external stream.

---

## Pattern 1 — Custom Data Source (Read)

The read function takes **no arguments** and returns either:
- A **list** → batch mode (all rows delivered at once)
- A **generator** (using `yield`) → streaming mode (rows delivered continuously)

### Minimal example — batch

```sql
CREATE EXTERNAL STREAM daily_prices (
    symbol string,
    price  float32,
    ts     datetime64(3)
)
AS
$$
import requests
from datetime import datetime

def fetch_prices():
    resp = requests.get('https://api.example.com/prices').json()
    rows = []
    for item in resp:
        rows.append((item['symbol'], item['price'], datetime.utcnow()))
    return rows
$$
SETTINGS
    type = 'python',
    read_function_name  = 'fetch_prices'
;
```

### Streaming example — generator / WebSocket

```sql
CREATE EXTERNAL STREAM live_feed (
    user_id  string,
    action   string,
    payload  string
)
AS
$$
import websocket, json

def stream_events():
    ws = websocket.create_connection('wss://feed.example.com/events')
    while True:
        msg = json.loads(ws.recv())
        yield (msg.get('user_id', ''), msg.get('action', ''), json.dumps(msg))
$$
SETTINGS
    type = 'python',
    read_function_name = 'stream_events'
;
```

Query the stream:

```sql
SELECT user_id, action, payload
FROM   live_feed
WHERE  action = 'purchase';
```

---

## Pattern 2 — Custom Data Sink (Write)

The write function is called **vectorized**: one Python `list` per column for the current batch of rows. Iterate using `zip(col1, col2, ...)`.

```sql
CREATE EXTERNAL STREAM webhook_sink (
    event_type string,
    payload    string
)
AS
$$
import requests

def _to_text(value):
    return value.decode('utf-8') if isinstance(value, (bytes, bytearray)) else value

def send_to_webhook(event_type, payload):
    for etype, body in zip(event_type, payload):
        requests.post(
            'https://hooks.example.com/ingest',
            json={'type': _to_text(etype), 'data': _to_text(body)}
        )
$$
SETTINGS
    type = 'python',
    write_function_name = 'send_to_webhook'
;
```

Insert into the sink from a streaming query:

```sql
INSERT INTO webhook_sink
SELECT event_type, to_string(raw)
FROM   source_stream
WHERE  severity = 'critical';
```

---

## Pattern 3 — Streaming Transform

For row-by-row or batch transformation, define a function that **accepts column lists** (vectorized) and **returns a list of output rows**. Then call it via `python_table()` in the `FROM` clause.

```sql
CREATE EXTERNAL STREAM my_transform (
    a int32,
    b int32,
    result int32          -- output column
)
AS
$$
def add_columns(a, b):
    return [(x + y,) for x, y in zip(a, b)]
$$
SETTINGS
    type = 'python',
;
```

Apply the transform to an existing stream:

```sql
SELECT result
FROM   python_table(my_transform, input_stream.a, input_stream.b)
EMIT   STREAM;
```

---

## Complete Real-World Example — Bluesky Jetstream

Streams every public post from the Bluesky social network's WebSocket firehose into Timeplus:

```sql
SYSTEM INSTALL PYTHON PACKAGE 'websockets';

CREATE EXTERNAL STREAM bluesky_posts (
    did         string,
    handle      string,
    text        string,
    created_at  string,
    langs       string,
    raw         string
)
AS
$$
import asyncio, json, websockets

JETSTREAM_URL = 'wss://jetstream2.us-east.bsky.network/subscribe?wantedCollections=app.bsky.feed.post'

def read_jetstream():
    import threading, queue

    q = queue.Queue(maxsize=1000)

    async def _ws_reader():
        async with websockets.connect(JETSTREAM_URL) as ws:
            async for raw_msg in ws:
                try:
                    msg = json.loads(raw_msg)
                    if msg.get('kind') != 'commit':
                        continue
                    commit  = msg.get('commit', {})
                    record  = commit.get('record', {})
                    if record.get('$type') != 'app.bsky.feed.post':
                        continue
                    did     = msg.get('did', '')
                    text    = record.get('text', '')
                    created = record.get('createdAt', '')
                    langs   = ','.join(record.get('langs', []))
                    q.put((did, '', text, created, langs, raw_msg))
                except Exception:
                    pass

    def _thread():
        asyncio.run(_ws_reader())

    t = threading.Thread(target=_thread, daemon=True)
    t.start()

    while True:
        yield q.get()
$$
SETTINGS
    type = 'python',
    read_function_name = 'read_jetstream';
```

Query examples:

```sql
-- Count posts per language in a 1-minute tumbling window
SELECT   langs, count() AS cnt
FROM     bluesky_posts
GROUP BY langs, window_start
EMIT     STREAM
SETTINGS tumble_window_interval = 60;

-- Search for a keyword in real-time
SELECT did, text, created_at
FROM   bluesky_posts
WHERE  position('timeplus' IN lower(text)) > 0;
```

---

## Function Signature Quick Reference

| Mode      | Signature                        | Returns                             |
|-----------|----------------------------------|-------------------------------------|
| Read      | `def fn() -> ...`                | `list` (batch) or `generator` (stream) |
| Write     | `def fn(col1, col2, ...)` → `None` | nothing; side-effects only        |
| Transform | `def fn(col1, col2, ...) -> list` | list of row tuples                 |

---

## Common Mistakes to Avoid

| Mistake | Fix |
|---|---|
| Using `async def` or `async for` | Not supported — use threads + `queue.Queue` to bridge async libs |
| Forgetting `type = 'python'` in SETTINGS | Always required |
| Returning a plain value instead of a tuple in multi-column transform | Each row must be a tuple: `(val1, val2)` |
| Treating incoming SQL strings as Python `str` on Enterprise 3.2.2+ | Decode `bytes` values first, including nested `array(string)` elements |
| Importing heavy packages at module level in a tight loop | Move imports outside the generator/function body |
| Installing packages after stream creation | Run `SYSTEM INSTALL PYTHON PACKAGE` before `CREATE EXTERNAL STREAM` |
| Putting credentials directly in the Python body | Put JSON credentials in a named collection under `init_function_parameters` and parse it in an init hook |
| Setting `init_function_parameters` without `init_function_name` | Define an init hook and set `init_function_name` |

---

## Tips for AI-Assisted Code Generation

When helping a user write a Python Table Function, always:

1. **Clarify the schema** — ask what columns are needed and their types before writing code.
2. **Identify the mode** — read (source), write (sink), or transform. A stream can support both read and write with separate function names.
3. **Check for async libraries** — if the user's target SDK is async (e.g., `aiohttp`, `websockets`), wrap it using `threading` + `queue.Queue` since async generators are not supported.
4. **Use `yield` for continuous data** — return a list only when the data set is finite and bounded.
5. **Keep state in module-level variables** — for stateful transforms, declare state outside the function with `global`.
6. **Test the schema match** — the number and order of columns returned by the function must exactly match the `CREATE EXTERNAL STREAM` column list.
7. **Use named collections for credentials** — pack secrets as JSON in the
   `init_function_parameters` collection key and parse them in an init hook.
8. **Decode incoming strings** — on Timeplus Enterprise 3.2.2+, write and
   transform functions receive SQL strings as Python `bytes`; decode them before
   string matching, regex, JSON object construction, or HTTP request payload
   generation.
