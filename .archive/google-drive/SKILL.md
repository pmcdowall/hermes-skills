---
name: google-drive
description: Access OFS Google Drive and Google Sheets via the Google APIs. Use when asked to read or update project sheets, find files in Drive, or update hosting records. Pre-configured for paul@onefellswoop.com.au's OFS Google Workspace Drive.
---

# Google Drive & Sheets Skill — OFS

Access Google Drive and Sheets via a service account (app-only auth).

## Credentials
- **Service account:** `rocky-agent@rocky-agent-493502.iam.gserviceaccount.com`
- **Key file:** `~/.openclaw/google-service-account.json`
- **1Password:** "Google Drive API — Rocky" (Paul's Vault)
- **Project:** `rocky-agent-493502`

## Setup (already done)
```python
from google.oauth2 import service_account
from googleapiclient.discovery import build

SCOPES = ['https://www.googleapis.com/auth/drive', 'https://www.googleapis.com/auth/spreadsheets']
creds = service_account.Credentials.from_service_account_file(
    '/home/ubuntu/.openclaw/google-service-account.json', scopes=SCOPES)
drive = build('drive', 'v3', credentials=creds)
sheets = build('sheets', 'v4', credentials=creds)
```

## Web Hosting Record Sheet Structure
Each project has one spreadsheet with a single sheet named **Template**.
Layout: Column B = label, Column C = value (data starts row 6).

Key rows:
| Row | Label |
|-----|-------|
| 8   | Client |
| 9   | Job number |
| 10  | Project name |
| 11  | Site name |
| 12  | **Status** (e.g. Live, Staging, Archived, Compromised) |
| 13  | Revision |
| 26  | ServerPilot app (EOI Staging) |
| 57+ | Full Staging Site section |
| 103+| Full Live Site section |

## Common Operations

### Read a value from a sheet
```python
result = sheets.spreadsheets().values().get(
    spreadsheetId=SHEET_ID,
    range='Template!C12'  # Status field
).execute()
value = result.get('values', [['']])[0][0]
```

### Update a value
```python
sheets.spreadsheets().values().update(
    spreadsheetId=SHEET_ID,
    range='Template!C12',
    valueInputOption='RAW',
    body={'values': [['Archived']]}
).execute()
```

### Search Drive for a file by name
```python
results = drive.files().list(
    q="name contains 'Europa' and mimeType='application/vnd.google-apps.spreadsheet'",
    fields='files(id, name)'
).execute()
```

### List all files in a folder
```python
results = drive.files().list(
    q=f"'{FOLDER_ID}' in parents",
    fields='files(id, name, mimeType)',
    pageSize=200
).execute()
```

## Known Sheet IDs (Staging Server Projects)
| Project | Sheet ID |
|---------|----------|
| Europa on Alma | `16cEX6f7pmwky9YTnDomFeryqfIYkbTeDwEZW0w8EWu4` |
| Showcase - Europa on Alma | `1JoMQYBFT78002NOy4Le5EaeueA-Mv1fMkoUBddSnibk` |
| Dux Churchill website | `13rSahZUZqLZstYNwGOQTcCG4xp2SxlcuRu9ZY5qAVeA` |
| Showcase - Dux Churchill | `1hO3kwD013Xb0KB2ZZqDf22mdjKIzzPIXL0q_zrl3ycg` |
| Bellevue Geelong website | `1up5LbdWoOK-PkAKz7DrVz_g90DBIudg9j6_VTgvr_8k` |
| SHC Bellbird website | `13yOShL2GBuKCAqIQV048LfzSMUKCNxqq_uQ_XDp25EY` |
| St Hedwig Village | `1i85XoQq34FcOjIA0LKoR9kNsYRqubs5CaTp-Ok3IdOA` |
| Showcase - Bellbird on High | `1QIpT4EwCJIslnHcajT4B7iGdTqQKCxhESGlbDIOPjN4` |
| McQuoin Park | `1yHPid9Ltj80SEpFh8WBCYufYuofmzLlFGx8HEm5sFB8` |
| McQuoin Park Stage 2 | `1Tzf8PMhlR65Za1EalOWBXq4sYky9IOBHYXKPsRj6NrM` |
| Showcase - McQuoin Park | `1eSzK4lRxYldN1SRt5kBgxCabzJJ5VPT2BswD_sKqSb0` |
| Retire Victoria website | `1GV69hduInoK296VL3JwrjlQ9fRKo-c L468DNUZiOgg4` |
| Retirement Your Way websites | `1H2H_cZLIdUmb06QOFehcZxdcdy8k3st7L--fde63VRw` |
| Retire At Southport website | `1wQbFRRXLl-yfbzV7tlu15zqQu1qXxwJyToYzYnz3isM` |
| Pinetree Retirement Village | `1n5XlvbGHh4BMDZ2KVslpew3wFTE9CDOPGcBeSE_zmI0` |
| Newgreens website | `1O9DvSFFrCNHcfv26iBKLjs74JVQizT4ywC7xlslIEzc` |
| Showcase - Callisto Place | `1_GrJD77imzSs46tmQ19sZQTxjUQA24Pp-k-ubX2Z3W0` |
| Showcase - The Alba | `1iIsBHQZJwAOUzCBDoS7BCxzDXwf9SNrIW9dm2B3SVX4` |
| Showcase - The Falls Estate | `1pABMpFdCfflT3BjZZJkF5S6xS4rTjp2N8ZJhJCY71gE` |
| The Alba website | `1nrIvH7WwYqqjp6KZBSuJtgewFnAZwDkjzf962kWaxyw` |

## Notes
- The service account has Editor access to the shared OFS Drive folder
- Python libraries required: google-auth, google-auth-httplib2, google-api-python-client (installed system-wide with --break-system-packages)
- Status values used: Live, Staging, Archived, Compromised, In Development
