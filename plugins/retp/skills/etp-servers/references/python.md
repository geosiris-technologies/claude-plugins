# The `retp` Python package

When the user wants a script (a notebook, numpy arrays, a loop over objects) rather than
command lines, the `retp` Python package wraps the same Rust client. Its page upstream is
`docs/python.md`. It is on PyPI (`pip install retp`: 0.2.0, 0.3.0, 0.4.0); install it in a
virtual environment of the user's, after asking.

```python
import os
import retp

# The token comes from where the user keeps it: never a literal in the script.
token = open(os.path.expanduser("~/.config/retp/demo.token")).read().strip()

with retp.connect("wss://etp.example.com/etp", token=token) as session:
    for space in session.dataspaces():
        print(space["path"], space["uri"])
    listed = session.resources("demo", depth=1)
    for resource in listed["resources"]:
        print(resource["uri"], resource["name"])
```

- `retp.connect(url, token=..., partition=..., insecure=..., writing=..., rest=...)`.
  `partition` is sent as the header `data-partition-id`; the other headers a server may want
  are not bridged yet (the command line has them). In 0.3.0 only a bearer `token` is bridged.
  From 0.4.0, without a token, the session logs in as the command line does, from `user`,
  `password`, `token_url`, `azure_tenant`, `cognito_region`, `client_id`, `client_secret`,
  `oauth_scope`. The bridge reads no environment variable and no catalogue: pass what the
  script read from the user's file or `os.environ`, never a literal.
- `rest=True` (0.4.0) talks to the OSDU Reservoir DDMS REST API, with its `https://` URL.
- `session.reauthorize(new_token)` (0.4.0) renews the token of an open session before it runs
  out; the session stays the same.
- A session writes only when opened with `writing=True`; `put_objects` and `push_epc` follow
  the same consent rule as `push` (section 3 of `SKILL.md`). From 0.4.0,
  `retp.plan_push("model.epc", "demo", ...)` says what `push_epc` would send, with no session
  (show it before asking), and `push_epc` takes the options of the command line
  (`skip_present`, `journal`, `split`, `reconnect`, `resume_session`): see `docs/python.md`
  upstream.
- Objects are dictionaries in the EnergyML JSON encoding (`obj["$type"]`); arrays come back as
  flat numpy arrays with their dimensions (`session.read_array(uri, path)`).
- Errors derive from `retp.RetpError`; plural reads return `(found, errors)` so nothing is
  silently missing. From 0.4.0, a session used by a second thread at once raises
  `SessionBusy` (one session per thread).

Do not print the token, store it in the notebook, or pass it in a URL. The bridge keeps it out
of `repr` and error messages; the script must too.
