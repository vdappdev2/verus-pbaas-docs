# z_getoperationstatus

> Check the status and result of asynchronous operations like `sendcurrency`.

**Category:** Multichain
**Daemon help:** `verus help z_getoperationstatus`

---

## Summary

`z_getoperationstatus` returns the current status, timing, and result for one or more async operations. Every call to [`sendcurrency`](sendcurrency.md) returns an operation ID (opid) rather than a transaction ID — poll this RPC to track progress and retrieve the resulting txid on success or error details on failure.

Operations remain in memory for the lifetime of the daemon process. Without arguments, returns all operations known to the node.

---

## Syntax

```
z_getoperationstatus (["operationid", ...])
```

---

## Parameters

| # | Name | Type | Required | Default | Description |
|---|------|------|----------|---------|-------------|
| 1 | `operationids` | array | No | all | Array of operation ID strings to query. Omit to return all operations. |

---

## Return value

Returns an array. Each element describes one operation:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | The operation ID (e.g., `"opid-2503d02c-ca1a-4083-83cc-0dc5911392d7"`). |
| `status` | string | Current status. See [Status lifecycle](#status-lifecycle). |
| `creation_time` | number | Unix timestamp when the operation was created. |
| `result` | object | Present on `"success"`. Contains `txid` — the broadcast transaction ID. |
| `error` | object | Present on `"failed"`. Contains `code` and `message` describing the failure. |
| `execution_secs` | number | Time spent executing, in seconds. |
| `method` | string | The RPC that created this operation (e.g., `"sendcurrency"`). |
| `params` | array | The parameters passed to the originating RPC call. |

### Status lifecycle

| Status | Description |
|--------|-------------|
| `"queued"` | Operation is waiting to execute. |
| `"executing"` | Operation is actively running (signing, building transaction). |
| `"success"` | Transaction was signed and broadcast. `result.txid` is available. |
| `"failed"` | Operation failed. `error.code` and `error.message` describe why. |

Operations move through `queued` → `executing` → `success` or `failed`. Most operations complete in under a second, so polling immediately after `sendcurrency` often returns `"success"` directly.

---

## Important behaviors

- **Operations persist in memory.** Completed operations (success or failed) remain queryable until the daemon restarts. There is no automatic cleanup.
- **No disk persistence.** If the daemon restarts, all operation history is lost. For durable records, save the txid from successful operations.
- **One result per sendcurrency call.** Even when `sendcurrency` batches multiple outputs, the operation produces a single txid.
- **`params` echoes the original request.** This is useful for correlating results when polling multiple operations — each result includes the method and parameters that created it.
- **Empty array means no operations.** A freshly started daemon with no async calls returns `[]`.

---

## Examples

### Poll a specific operation

```
z_getoperationstatus '["opid-2503d02c-ca1a-4083-83cc-0dc5911392d7"]'
```

```json
[
  {
    "id": "opid-2503d02c-ca1a-4083-83cc-0dc5911392d7",
    "status": "success",
    "creation_time": 1774979757,
    "result": {
      "txid": "dfbd50e309ffca603174b9028dda698bd9ac15cfc45657c3183afdcbc30b1316"
    },
    "execution_secs": 0.07735,
    "method": "sendcurrency",
    "params": [
      {
        "address": "test1.mcp3@",
        "amount": 1,
        "currency": "mcp3"
      }
    ]
  }
]
```

The operation completed in ~77ms. The resulting transaction can now be inspected with [`gettransaction`](gettransaction.md).

### Poll all operations

```
z_getoperationstatus
```

Returns every operation since the daemon started — useful for finding an opid you didn't save, or checking if any operations failed.

### Conversion operation

After sending a conversion via [`sendcurrency`](sendcurrency.md):

```
sendcurrency "third.testidx@" '[{"currency":"vrsctest","amount":0.1,"convertto":"kaiju","address":"third.testidx@"}]'
```

Result: `"opid-b4fbdd77-7b78-4ee5-9c90-116e810b747b"`

Polling:

```json
{
  "id": "opid-b4fbdd77-7b78-4ee5-9c90-116e810b747b",
  "status": "success",
  "creation_time": 1774979768,
  "result": {
    "txid": "c6cef6a28988caffc2e5cee83a0088f7892413df4a5189d523a8a299a22e3217"
  },
  "execution_secs": 0.060329,
  "method": "sendcurrency",
  "params": [
    {
      "address": "third.testidx@",
      "amount": 0.1,
      "currency": "vrsctest",
      "convertto": "kaiju"
    }
  ]
}
```

The `params` field confirms this was a conversion operation. The txid can be used with [`gettransaction`](gettransaction.md) to see the `reservetransfer` details and conversion output.

### Typical polling pattern

```python
import time, json, subprocess

def poll_operation(opid, timeout=60):
    start = time.time()
    while time.time() - start < timeout:
        result = json.loads(subprocess.check_output(
            ["verus", "z_getoperationstatus", json.dumps([opid])]
        ))
        if not result:
            raise Exception("Operation not found")
        op = result[0]
        if op["status"] == "success":
            return op["result"]["txid"]
        if op["status"] == "failed":
            raise Exception(op["error"]["message"])
        time.sleep(1)
    raise Exception("Timeout")
```

In practice, most operations complete in under a second. A single poll immediately after `sendcurrency` is usually sufficient.

---

## See also

- [`sendcurrency`](sendcurrency.md) — the primary RPC that produces operation IDs
- [`gettransaction`](gettransaction.md) — inspect the resulting transaction after success
- [`listtransactions`](listtransactions.md) — view transaction history including conversions
