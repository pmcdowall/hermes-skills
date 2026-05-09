---
name: wiplist
description: Access WIPlist for One Fell Swoop via the WIPlist REST API. Use when asked about jobs, estimates, invoices, purchases, time entries, tasks, contacts, or team schedules. Covers: listing/searching jobs, checking estimate status, reviewing invoices, logging time, querying team members, and financial summaries. Pre-configured for paul@onefellswoop.com.au.
---

# WIPlist Skill — OFS Studio Agent

Access WIPlist via its private REST API. Always fetch a fresh token at the start of each session — tokens last 7 days but fetch fresh to be safe.

## Credentials

```
Base URL:    https://app.wiplist.io
Solution ID: 4revg6
Email:       paul@onefellswoop.com.au
Password:    jmn3TQG3hea7vbpymx
```

## Authentication

```bash
TOKEN=$(curl -s -X POST "https://app.wiplist.io/api/v1/login" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-Solution: 4revg6" \
  -d '{"email":"paul@onefellswoop.com.au","password":"jmn3TQG3hea7vbpymx"}' \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['data']['token'])")
```

Always include on every request:
- `Authorization: Bearer $TOKEN`
- `X-Solution: 4revg6`
- `Accept: application/json`

## Key Endpoints

### Jobs
```bash
# Search active jobs
curl -s "https://app.wiplist.io/api/v1/jobs?filter[job_status]=200,210,215,220&per_page=50&include=company,contact,manager,tasks" \
  -H "Authorization: Bearer $TOKEN" -H "X-Solution: 4revg6" -H "Accept: application/json"

# Search by job number or title
curl -s "https://app.wiplist.io/api/v1/jobs?filter[job_number_or_title]={query}&filter[job_status]=200,210,215,220&per_page=20&simple=true" \
  -H "Authorization: Bearer $TOKEN" -H "X-Solution: 4revg6" -H "Accept: application/json"

# Single job detail
curl -s "https://app.wiplist.io/api/v1/jobs/{id}" \
  -H "Authorization: Bearer $TOKEN" -H "X-Solution: 4revg6" -H "Accept: application/json"
```

Job status codes: `200`=active, `210`=in progress, `215`=review, `220`=delivered, `300`=archived

### Estimates
```bash
# List estimates (optionally filter by job)
curl -s "https://app.wiplist.io/api/v1/estimates?per_page=20&filter[job_id]={jobId}" \
  -H "Authorization: Bearer $TOKEN" -H "X-Solution: 4revg6" -H "Accept: application/json"
```
Estimate status: `100`=draft, `200`=sent, `300`=approved, `400`=declined

### Invoices
```bash
curl -s "https://app.wiplist.io/api/v1/invoices?per_page=20&filter[job_id]={jobId}" \
  -H "Authorization: Bearer $TOKEN" -H "X-Solution: 4revg6" -H "Accept: application/json"
```
Key fields: `invoice_number`, `status`, `amount_total`, `amount_paid`, `amount_balance`, `date_due`, `is_sent`

### Time Entries
```bash
# List time entries for a user
curl -s "https://app.wiplist.io/api/v1/times?include=job,company&filter[user_id]={userId}&per_page=200" \
  -H "Authorization: Bearer $TOKEN" -H "X-Solution: 4revg6" -H "Accept: application/json"

# Submit a time entry
curl -s -X POST "https://app.wiplist.io/api/v1/times" \
  -H "Authorization: Bearer $TOKEN" -H "X-Solution: 4revg6" \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -d '{"user_id":"{userId}","job_id":"{jobId}","rate_id":"{rateId}","description":"Work description","date_start":"2026-04-03T09:00:00+00:00","date_end":"2026-04-03T10:00:00+00:00","time_total":3600,"type":200,"status":200,"unit_price":0,"unit_cost":0,"exchange_rate":1}'
```
`type`: `200`=billable, `300`=non-billable. `time_total` is in seconds.

### Purchases / POs
```bash
curl -s "https://app.wiplist.io/api/v1/purchases?per_page=20&filter[job_id]={jobId}" \
  -H "Authorization: Bearer $TOKEN" -H "X-Solution: 4revg6" -H "Accept: application/json"
```

### Tasks
```bash
curl -s "https://app.wiplist.io/api/v1/tasks?filter[job_id]={jobId}&per_page=50" \
  -H "Authorization: Bearer $TOKEN" -H "X-Solution: 4revg6" -H "Accept: application/json"
```

### Companies (Clients)
```bash
curl -s "https://app.wiplist.io/api/v1/companies?per_page=50" \
  -H "Authorization: Bearer $TOKEN" -H "X-Solution: 4revg6" -H "Accept: application/json"
```

### Contacts
```bash
curl -s "https://app.wiplist.io/api/v1/contacts?per_page=50&filter[company_id]={companyId}" \
  -H "Authorization: Bearer $TOKEN" -H "X-Solution: 4revg6" -H "Accept: application/json"
```

### Users (Team)
```bash
curl -s "https://app.wiplist.io/api/v1/users?per_page=50" \
  -H "Authorization: Bearer $TOKEN" -H "X-Solution: 4revg6" -H "Accept: application/json"
```

## Response Format

All responses follow: `{ "success": bool, "message": str, "data": [...], "meta": { pagination } }`

Pagination: `meta.pagination` contains `total`, `per_page`, `current_page`, `last_page`.

## Key Job Fields

`id`, `job_number`, `job_title`, `status`, `date_due`, `date_expected`, `budget`, `estimate_sale`, `invoice_sale`, `time_committed`, `time_committed_sale`, `is_scheduled`, `is_archived`, `manager` (obj), `company` (obj), `tasks` (array)

## See Also

- `references/ofs-team.md` — team member IDs, names, emails, roles
- `references/ofs-clients.md` — client/company IDs (populate on first use)
