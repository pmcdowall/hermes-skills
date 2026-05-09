---
name: salesforce
description: Access One Fell Swoop's Salesforce Sales Cloud instance. Use when asked to query data, build or review flows, write Apex, manage users, create reports/dashboards, troubleshoot issues, or work with integrations (Campaign Monitor, Twilio, Conga, Adobe Sign). Always test changes in sandbox before production.
---

# Salesforce Skill — One Fell Swoop

Full API access to the OFS Salesforce Sales Cloud instance via JWT Bearer authentication.

## Credentials

Stored in 1Password — Paul's Vault — "Salesforce — Rocky" (id: qnnsneznaewugz4ip3ffcjvzya)

```
Org: One Fell Swoop Pty Ltd (Enterprise Edition, AUS76)
Org ID: 00D90000000zGSpEAM
Production URL: https://onefellswoop.my.salesforce.com
Username: developer@onefellswoop.com.au
Auth: JWT Bearer Flow (RSA-256)
API Version: v59.0
```

## Authentication

```python
import subprocess, json, time, base64, urllib.request, urllib.parse
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding

def sf_token():
    """Get a fresh Salesforce access token via JWT Bearer Flow."""
    def op(f): return subprocess.check_output(
        ['op','item','get','qnnsneznaewugz4ip3ffcjvzya','--vault',"Paul's Vault",'--fields',f,'--reveal'],
        text=True).strip()
    
    pk_pem      = op('Private Key (PEM)')
    consumer_key = op('Production Consumer Key')
    username    = 'developer@onefellswoop.com.au'
    
    header  = base64.urlsafe_b64encode(json.dumps({"alg":"RS256","typ":"JWT"}).encode()).rstrip(b'=').decode()
    payload = base64.urlsafe_b64encode(json.dumps({
        "iss": consumer_key, "sub": username,
        "aud": "https://login.salesforce.com",
        "exp": int(time.time()) + 300
    }).encode()).rstrip(b'=').decode()
    
    pk  = serialization.load_pem_private_key(pk_pem.encode(), password=None)
    sig = base64.urlsafe_b64encode(
        pk.sign(f"{header}.{payload}".encode(), padding.PKCS1v15(), hashes.SHA256())
    ).rstrip(b'=').decode()
    
    with urllib.request.urlopen(urllib.request.Request(
        'https://login.salesforce.com/services/oauth2/token',
        data=urllib.parse.urlencode({
            'grant_type': 'urn:ietf:params:oauth:grant-type:jwt-bearer',
            'assertion': f"{header}.{payload}.{sig}"
        }).encode(),
        headers={'Content-Type': 'application/x-www-form-urlencoded'}
    ), timeout=15) as r:
        resp = json.load(r)
    
    return resp['access_token'], resp['instance_url']

TOKEN, INSTANCE = sf_token()
```

## REST API Calls

```python
def sf_get(path):
    req = urllib.request.Request(
        f"{INSTANCE}{path}",
        headers={'Authorization': f'Bearer {TOKEN}', 'Accept': 'application/json'}
    )
    with urllib.request.urlopen(req, timeout=15) as r:
        return json.load(r)

def sf_post(path, body):
    data = json.dumps(body).encode()
    req = urllib.request.Request(
        f"{INSTANCE}{path}", data=data, method='POST',
        headers={'Authorization': f'Bearer {TOKEN}', 'Content-Type': 'application/json', 'Accept': 'application/json'}
    )
    with urllib.request.urlopen(req, timeout=15) as r:
        return json.load(r)

def sf_patch(path, body):
    data = json.dumps(body).encode()
    req = urllib.request.Request(
        f"{INSTANCE}{path}", data=data, method='PATCH',
        headers={'Authorization': f'Bearer {TOKEN}', 'Content-Type': 'application/json'}
    )
    with urllib.request.urlopen(req, timeout=15) as r:
        return r.read()

def sf_query(soql):
    return sf_get(f"/services/data/v59.0/query?q={urllib.parse.quote(soql)}")

def sf_tooling(soql):
    return sf_get(f"/services/data/v59.0/tooling/query?q={urllib.parse.quote(soql)}")
```

## Common Operations

### SOQL Query
```python
result = sf_query("SELECT Id, Name, Email FROM Contact WHERE Email LIKE '%onefellswoop%' LIMIT 10")
for r in result['records']:
    print(r['Name'], r['Email'])
```

### Get Object Schema
```python
schema = sf_get("/services/data/v59.0/sobjects/Contact/describe/")
fields = [(f['name'], f['type'], f['label']) for f in schema['fields']]
```

### Create Record
```python
# Always confirm with Paul before creating records
new = sf_post("/services/data/v59.0/sobjects/Contact/", {
    "FirstName": "Test", "LastName": "Contact", "Email": "test@example.com"
})
print(new['id'])
```

### Update Record
```python
sf_patch(f"/services/data/v59.0/sobjects/Contact/{contact_id}/", {"Email": "new@example.com"})
```

### Deployment Rules (CRITICAL)
- **Sandbox:** Use Tooling API MetadataContainer (container name ≤ 32 chars)
- **Production:** Use Metadata REST API zip deploy — Tooling API is blocked in production
- **Never** use `sf org login jwt` — Python JWT Bearer flow works; sf CLI has PEM format issues
- **Always** use `testLevel: RunSpecifiedTests` in production — not RunLocalTests
- **Always** include test class with ≥75% coverage for any new/changed component
- **Exclude** from test list: `RG_ProductInfoctr_Test`, `RG_ReportSummaryctr_Test`
- Full deployment guide: `Salesforce/RG_phase1/DEPLOYMENT_LEARNINGS.md`

### Tooling API (Apex, Flows, Metadata)
```python
# List Apex classes
classes = sf_tooling("SELECT Id, Name, Status, LengthWithoutComments FROM ApexClass ORDER BY Name")

# Execute anonymous Apex (sandbox only unless confirmed)
exec_result = sf_post("/services/data/v59.0/tooling/executeAnonymous/", {
    "anonymousBody": "System.debug('Hello from Rocky');"
})

# Get flow metadata
flows = sf_tooling("SELECT Id, ApiName, Label, Status, ProcessType FROM FlowDefinition ORDER BY Label")
```

### Run a Report
```python
# List reports
reports = sf_query("SELECT Id, Name, FolderName FROM Report ORDER BY Name LIMIT 20")

# Run a report
report_result = sf_get(f"/services/data/v59.0/analytics/reports/{report_id}")
```

## Installed Integrations
| Package | Purpose |
|---------|---------|
| Campaign Monitor for Salesforce | Email marketing sync (Beaufort) |
| Conga Composer | Document generation |
| Adobe Acrobat Sign | E-signatures |
| ActivityHub | Activity tracking |
| b2bmaIntegration | B2B marketing automation |
| HubSpot Integration | HubSpot sync |
| FlowActionsBasePack | Flow utilities |
| FlowScreenComponentsBasePack | Flow screen components |
| Mass Edit from List Views | Bulk editing |

## Data Summary (as of 2026-04-09)
- Accounts: 77
- Contacts: 38,497
- Leads: 32,593
- Opportunities: 33,400

## Sandbox

### Sandbox: Rocky
- **License:** Developer Pro
- **Storage:** 2GB
- **Based on:** Production
- **Location:** Hyperforce AUS18S
- **Org ID:** 00DBm00000AHMPl
- **Login URL:** https://onefellswoop--rocky.sandbox.my.salesforce.com/
- **Username:** developer@onefellswoop.com.au.rocky
- **External Client App Name:** Rocky AI Assistant Sandbox
- **API Name:** Rocky_AI_Assistant_Sandbox
- **Contact Email:** developer@onefellswoop.com.au
- **Consumer Key:** 3MVG9vVKBifrxMjbFaMlE45w.x4LYaXh3ZqBGF2OxGWQOrA7VXF5yHbCR0d2wgpDlLi3NvzaJgizmXLRPZKSf
- **Consumer Secret (1PW):** store in 1Password — Consumer Secret field on "Salesforce — Rocky"
- **Certificate file (on Paul's Mac):** /Users/paulm/Projects/openclaw/Salesforce integration/Rocky_AI_Assistant_Rocky.crt
- **Auth:** JWT Bearer Flow (RSA-256) — same Private Key as production, different Consumer Key
- **Sandbox Login URL for JWT aud:** https://test.salesforce.com

### Sandbox Auth Code

```python
def sf_token_sandbox():
    """Get a fresh Salesforce access token for the Rocky sandbox."""
    def op(f): return subprocess.check_output(
        ['op','item','get','qnnsneznaewugz4ip3ffcjvzya','--vault',"Paul's Vault",'--fields',f,'--reveal'],
        text=True).strip()
    
    pk_pem       = op('Private Key (PEM)')
    consumer_key = '3MVG9vVKBifrxMjbFaMlE45w.x4LYaXh3ZqBGF2OxGWQOrA7VXF5yHbCR0d2wgpDlLi3NvzaJgizmXLRPZKSf'
    username     = 'developer@onefellswoop.com.au.rocky'
    
    header  = base64.urlsafe_b64encode(json.dumps({"alg":"RS256","typ":"JWT"}).encode()).rstrip(b'=').decode()
    payload = base64.urlsafe_b64encode(json.dumps({
        "iss": consumer_key, "sub": username,
        "aud": "https://test.salesforce.com",
        "exp": int(time.time()) + 300
    }).encode()).rstrip(b'=').decode()
    
    pk  = serialization.load_pem_private_key(pk_pem.encode(), password=None)
    sig = base64.urlsafe_b64encode(
        pk.sign(f"{header}.{payload}".encode(), padding.PKCS1v15(), hashes.SHA256())
    ).rstrip(b'=').decode()
    
    with urllib.request.urlopen(urllib.request.Request(
        'https://test.salesforce.com/services/oauth2/token',
        data=urllib.parse.urlencode({
            'grant_type': 'urn:ietf:params:oauth:grant-type:jwt-bearer',
            'assertion': f"{header}.{payload}.{sig}"
        }).encode(),
        headers={'Content-Type': 'application/x-www-form-urlencoded'}
    ), timeout=15) as r:
        resp = json.load(r)
    
    return resp['access_token'], resp['instance_url']

TOKEN, INSTANCE = sf_token_sandbox()  # Use this when working in sandbox
```

- **Rule:** ALL Apex, Flow, and metadata changes must be tested in sandbox first

## Safety Rules
1. READ operations (SOQL, schema, reports): always safe, go ahead
2. WRITE (create/update): describe what will happen, confirm with Paul
3. BULK operations (>10 records): always confirm
4. DESTRUCTIVE (delete, truncate): never without explicit approval + sandbox test
5. APEX/FLOW deploy to production: sandbox test required first
6. Never modify other users' permissions without explicit instruction

## API Limits Reference
- Daily API calls: 15,000 (Enterprise Edition standard)
- Bulk API: 10,000 records/batch
- Tooling API: available for metadata operations
- Streaming API: available for real-time data (PushTopic, GenericStreaming)
