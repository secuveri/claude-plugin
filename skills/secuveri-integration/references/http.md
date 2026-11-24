# Plain HTTP (any language)

No SDK needed: three HTTPS calls plus two signature checks. The Python below uses `requests`, `PyJWT[crypto]` and the standard library; port the same steps to any language. Call `get_code_sample` on the MCP server for its current Node.js and Python samples.

## Start a session

```python
import os, requests

API = "https://api-verify.secuveri.com"
HEADERS = {"x-api-key": os.environ["SECUVERI_API_KEY"]}

def start_session(order):
    response = requests.post(
        f"{API}/v1/sessions",
        headers={**HEADERS, "Idempotency-Key": f"age-check-{order.id}"},
        json={"workflow_id": os.environ["SECUVERI_WORKFLOW_ID"], "vendor_data": order.id},
        timeout=15,
    )
    response.raise_for_status()
    session = response.json()
    save_verification_id(order.id, session["verification_id"])
    return session["url"]  # redirect the person here
```

## Read and verify the signed result

```python
import time, jwt  # PyJWT

_jwks = {"keys": None, "at": 0.0}

def _signing_key(kid):
    for refresh in (False, True):
        if refresh or not _jwks["keys"] or time.time() - _jwks["at"] > 300:
            _jwks["keys"] = requests.get(f"{API}/.well-known/jwks.json", timeout=10).json()["keys"]
            _jwks["at"] = time.time()
        for key in _jwks["keys"]:
            if key.get("kid") == kid:
                return jwt.PyJWK(key)
    raise ValueError("unknown signing key")

def get_result(verification_id):
    response = requests.get(f"{API}/v1/verifications/{verification_id}/result", headers=HEADERS, timeout=15)
    if response.status_code == 409:
        return None  # not finished yet
    response.raise_for_status()
    token = response.json()["result"]
    kid = jwt.get_unverified_header(token)["kid"]
    payload = jwt.decode(
        token,
        _signing_key(kid),
        algorithms=["ES256"],
        audience=os.environ["SECUVERI_ORGANIZATION_ID"],
    )
    return payload, token  # store token as evidence
```

Then apply the gate rules from SKILL.md to `payload["verification"]`.

## Verify a webhook

```python
import hmac, hashlib

def verify_webhook(raw_body: bytes, header: str, secret: str, tolerance=300):
    parts = [p.split("=", 1) for p in (header or "").split(",") if "=" in p]
    timestamps = [v for k, v in parts if k == "t"]
    signatures = [v for k, v in parts if k == "v1"]
    if not timestamps or not signatures:
        raise ValueError("malformed secuveri-signature")
    t = timestamps[0]
    if abs(time.time() - int(t)) > tolerance:
        raise ValueError("stale webhook")
    expected = hmac.new(secret.encode(), f"{t}.".encode() + raw_body, hashlib.sha256).hexdigest()
    if not any(hmac.compare_digest(expected, s) for s in signatures):
        raise ValueError("bad signature")
```

Pass the raw request body (Flask `request.get_data()`, Django `request.body`, FastAPI `await request.body()`), never re-serialised JSON.
