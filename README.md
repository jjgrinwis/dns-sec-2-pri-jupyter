# Convert an EdgeDNS Secondary Zone to Primary

This project contains a Jupyter notebook for converting one or more Akamai EdgeDNS zones from `SECONDARY` to `PRIMARY`.

The notebook uses the EdgeDNS API to keep the existing records in place. It does not export and re-import the DNS records manually.

The default example operates on `nipshotnews.nl`. Review the zone list before running any cell that changes Akamai configuration.

## What The Notebook Does

The notebook is intentionally split into separate cells so each potentially disruptive step can be reviewed and run independently:

1. Creates an authenticated Akamai EdgeGrid session from `~/.edgerc`.
2. Checks that every configured zone is `SECONDARY` and has an eligible activation state: `ACTIVE` or `ERROR`.
3. Checks the DNSSEC transition state before conversion:
   - Queries the parent zone's authoritative name server and Google Public DNS (`8.8.8.8`) for `DS` records.
   - Queries an EdgeDNS authoritative name server and Google Public DNS for `DNSKEY` records.
   - Stops if the parent still publishes a DS record or if EdgeDNS still serves DNSKEY records.
4. Submits the asynchronous EdgeDNS convert-to-primary request.
5. After the conversion completes, checks that the zone is `PRIMARY` and `ACTIVE`.
6. Enables EdgeDNS DNSSEC Sign and Serve with `ECDSA_P256_SHA256` by default.
7. Retrieves the new DS record from the EdgeDNS DNSSEC status API so it can be submitted to the registrar.

The conversion request is asynchronous. The notebook does not poll for completion automatically; rerun the relevant validation or follow-up cell after Akamai finishes the operation.

## Requirements

- macOS, Linux, or another environment with Python support.
- Python 3.13 or newer. The project pins the development interpreter through `.python-version`.
- An Akamai API client with EdgeDNS permissions.
- An Akamai EdgeGrid credentials file at `~/.edgerc`.
- `dig`, used by the DNSSEC preflight checks. It is normally available on macOS and most Linux distributions.
- Network access to Akamai APIs, the zone's authoritative name servers, and Google Public DNS.

## Install `uv`

`uv` is the Python project and environment manager used by this repository. Choose one installation method.

### macOS with Homebrew

```bash
brew install uv
```

### Official installer

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Restart the terminal or load the shell configuration updated by the installer, then verify the installation:

```bash
uv --version
```

On macOS, install `dig` if it is not already available:

```bash
brew install bind
```

Verify it with:

```bash
dig -v
```

## Install The Project

From the repository directory, run:

```bash
uv sync
```

This creates or updates the project virtual environment in `.venv` and installs the dependencies from `pyproject.toml` and `uv.lock`, including:

- `edgegrid-python` for Akamai EdgeGrid authentication.
- `jupyter` for running the notebook.

To verify the environment without activating it:

```bash
uv run python --version
uv run jupyter --version
```

The project requires Python 3.13 or newer. If `uv` cannot find a suitable interpreter, install one with:

```bash
uv python install 3.13
uv sync
```

## Configure Akamai Credentials

Create or obtain an EdgeGrid credentials file at:

```text
~/.edgerc
```

The selected section must contain at least the EdgeGrid `host`, `client_token`, `client_secret`, and `access_token` values. Follow Akamai's [EdgeGrid authentication setup documentation](https://techdocs.akamai.com/developer/docs/set-up-authentication-credentials) to create credentials with the required EdgeDNS permissions.

The notebook reads these environment variables:

| Variable                    | Required | Description                                                       |
| --------------------------- | -------- | ----------------------------------------------------------------- |
| `AKAMAI_EDGEGRID_SECTION`   | No       | The `~/.edgerc` section to use. Defaults to `default`.            |
| `AKAMAI_ACCOUNT_SWITCH_KEY` | No       | Account Switch Key used when operating on another Akamai account. |

Example:

```bash
export AKAMAI_EDGEGRID_SECTION=default
export AKAMAI_ACCOUNT_SWITCH_KEY="A-B-1YX7F123:1-2ABCD"
```

Do not put credentials, access tokens, client secrets, or account switch keys in the notebook or in `README.md`. Do not commit `~/.edgerc`.

## Open The Notebook

Start JupyterLab through the project environment:

```bash
uv run jupyter lab
```

Open `convert-2-primary.ipynb` in the browser.

Alternatively, start the classic Jupyter Notebook interface:

```bash
uv run jupyter notebook
```

To open the notebook directly without relying on the browser's file picker:

```bash
uv run jupyter lab convert-2-primary.ipynb
```

Keep the terminal running while using Jupyter. Stop the server with `Ctrl-C` when finished.

### VS Code

Install the Jupyter extension in VS Code, open `convert-2-primary.ipynb`, and select the Python interpreter from this project:

```text
.venv/bin/python
```

The environment can be created or refreshed with `uv sync` before selecting it. VS Code may also discover the environment automatically.

## Configure The Zone List

In the `Validate state of zones` cell, edit `zone_names`:

```python
zone_names = [
	"nipshotnews.nl",
	# "another-secondary-zone.example",
]
```

All zones in a single conversion request must belong to the same contract and group according to the [Akamai convert-primary API documentation](https://techdocs.akamai.com/edge-dns/reference/post-zones-convert-primary-requests).

## DNSSEC Migration Order

The hidden primary's DNSSEC keys cannot be moved to EdgeDNS. If the zone currently uses DNSSEC, use this order:

1. Remove the zone's DS record at the registrar.
2. Wait for the DS record's TTL to expire, then confirm that the parent authoritative name servers and Google Public DNS no longer return the DS record.
3. Disable DNSSEC on the hidden primary, increment the SOA serial, and wait for EdgeDNS to transfer the unsigned zone.
4. Run the notebook's DNSSEC preflight cell. It must find no parent DS records and no DNSKEY records served by EdgeDNS.
5. Submit the convert-to-primary request.
6. Wait until the zone reports `PRIMARY` and `ACTIVE`.
7. Run the `Enable DNSSEC Sign and Serve` cell. The default algorithm is `ECDSA_P256_SHA256`.
8. Run the `Get the new DS record` cell and submit the returned DS record to the registrar.

The DS check is intentionally a guard: if it finds a DS record, the notebook stops before conversion. A stale DS record combined with an unsigned zone can cause validating resolvers to return `SERVFAIL`.

If the zone is already unsigned, the DS and DNSKEY checks should pass without changing the hidden primary.

## Run The Cells In Order

Run the cells in this order:

1. **Set Up EdgeGrid Session**: confirms the credentials file and configures the authenticated session.
2. **Validate state of zones**: confirms zone type and activation state.
3. **Check if DNSSEC is still enabled**: checks DS and DNSKEY records. This cell requires `dig`.
4. **Convert zone**: submits the asynchronous conversion request.
5. Wait for the request to complete, then run **Enable DNSSEC Sign and Serve**. It refuses to proceed until each zone is `PRIMARY` and `ACTIVE`.
6. **Get the new DS record**: retrieves the DS value generated by EdgeDNS.

Do not rerun the conversion cell casually. A successful request changes the zone workflow and the operation is not a harmless read-only check.

## Troubleshooting

### `Could not find .edgerc`

Create `~/.edgerc`, or verify that the notebook is running as the same user that owns the file:

```bash
ls -l ~/.edgerc
```

### `401 Unauthorized` or `403 Forbidden`

Check the selected `AKAMAI_EDGEGRID_SECTION`, credentials, account switch key, contract access, and EdgeDNS permissions. The account switch key must be available to the API client being used.

### Zone is `PENDING`

Wait for the current EdgeDNS activation or transfer to finish, then rerun the validation cell. Conversion is allowed only for `ACTIVE` or `ERROR` according to the API response used by this notebook.

### A DS record is still returned

The parent delegation is still signed or the old DS record is still cached. Confirm it has been removed at the registrar and wait at least the previous DS TTL before proceeding.

### DNSKEY records are still returned

DNSSEC is still enabled on the hidden primary, or EdgeDNS has not yet transferred the unsigned version. Disable DNSSEC on the hidden primary, bump the SOA serial, allow the transfer to complete, and rerun the DNSSEC preflight.

### `SignAndServe cannot be set without providing a SignAndServeAlgorithm`

Set `sign_and_serve_algorithm` in the enablement cell to one of the algorithms supported by your EdgeDNS account. The default in this notebook is `ECDSA_P256_SHA256`.

### No DS record is generated after enabling EdgeDNS DNSSEC

Key generation can take a moment. Rerun the `Get the new DS record` cell after the zone is active. The API may report a `PARENT_DS_MISSING` alert until the generated DS record has been published at the registrar.

### `dig` is missing

Install it with Homebrew on macOS:

```bash
brew install bind
```

## API References

- [Akamai EdgeDNS getting started](https://techdocs.akamai.com/edge-dns/reference/get-started)
- [Get a zone's settings](https://techdocs.akamai.com/edge-dns/reference/get-zone)
- [Submit a bulk zone convert-primary request](https://techdocs.akamai.com/edge-dns/reference/post-zones-convert-primary-requests)
- [Update a zone's settings](https://techdocs.akamai.com/edge-dns/reference/put-zone)
- [Get DNSSEC status](https://techdocs.akamai.com/edge-dns/reference/post-zones-dns-sec-status)
- [Akamai EdgeGrid authentication](https://techdocs.akamai.com/developer/docs/set-up-authentication-credentials)

## Project Files

- `convert-2-primary.ipynb`: the migration notebook.
- `pyproject.toml`: project metadata and dependencies.
- `uv.lock`: reproducible dependency lock file.
- `.python-version`: the expected Python version managed by `uv`.
- `main.py`: the default Python entry point generated for the project; it is not used by the notebook.
