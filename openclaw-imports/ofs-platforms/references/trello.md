---
name: trello
description: Interact with Trello boards, lists, and cards via the Trello REST API. Use when asked to view projects, check card status, move cards between lists, add comments, create cards, or produce project/workflow summaries from Trello. This skill is pre-configured for One Fell Swoop (OFS) studio workflow boards.
---

# Trello Skill

Interact with Trello via REST API calls using `exec` (curl). Credentials are pre-configured for OFS.

## Credentials

Stored in 1Password — Paul's Vault — "Trello API — Rocky" (id: nfwjhq446tnme7o2qqrk4bevni)

```
API Key:  791f899c1b00dea6f4b66f7f33598026  (non-secret, safe to keep here)
Base URL: https://api.trello.com/1
```

Fetch token at runtime:
```bash
TRELLO_KEY="791f899c1b00dea6f4b66f7f33598026"
TRELLO_TOKEN=$(op item get nfwjhq446tnme7o2qqrk4bevni --vault "Paul's Vault" --fields Token --reveal 2>/dev/null)
```

Always append `?key=$TRELLO_KEY&token=$TRELLO_TOKEN` (or `&key=...&token=...`) to every request.

## Key Board & List IDs

See `references/ofs-boards.md` for full board/list/member reference.

**Studio workflow** board: `690a62fbe5a80149bc54eedc`
**Studio allocations** board: `691ff75e6c3bb2055059cf55`

## Common Operations

### Get cards on a board (open only)
```bash
curl -s "https://api.trello.com/1/boards/{boardId}/cards?filter=open&fields=name,idList,due,dueComplete,shortUrl&key=KEY&token=TOKEN"
```

### Get cards in a specific list
```bash
curl -s "https://api.trello.com/1/lists/{listId}/cards?fields=name,due,dueComplete,shortUrl,idMembers&key=KEY&token=TOKEN"
```

### Get a single card (with description and comments)
```bash
curl -s "https://api.trello.com/1/cards/{cardId}?fields=name,desc,due,dueComplete,idList,shortUrl&key=KEY&token=TOKEN"
# With comments:
curl -s "https://api.trello.com/1/cards/{cardId}/actions?filter=commentCard&key=KEY&token=TOKEN"
```

### Create a card
```bash
curl -s -X POST "https://api.trello.com/1/cards" \
  -H "Content-Type: application/json" \
  -d '{"idList":"{listId}","name":"{cardName}","desc":"{description}","due":"{ISO8601}"}' \
  -G --data-urlencode "key=KEY" --data-urlencode "token=TOKEN"
# Simpler form:
curl -s -X POST "https://api.trello.com/1/cards?key=KEY&token=TOKEN&idList={listId}&name={name}"
```

### Move a card to a different list
```bash
curl -s -X PUT "https://api.trello.com/1/cards/{cardId}?key=KEY&token=TOKEN&idList={newListId}"
```

### Add a comment to a card
```bash
curl -s -X POST "https://api.trello.com/1/cards/{cardId}/actions/comments?key=KEY&token=TOKEN" \
  --data-urlencode "text=Your comment here"
```

### Set / update due date
```bash
curl -s -X PUT "https://api.trello.com/1/cards/{cardId}?key=KEY&token=TOKEN&due={ISO8601orNull}"
```

### Mark due date complete
```bash
curl -s -X PUT "https://api.trello.com/1/cards/{cardId}?key=KEY&token=TOKEN&dueComplete=true"
```

### Search cards across all boards
```bash
curl -s "https://api.trello.com/1/search?query={term}&modelTypes=cards&card_fields=name,idList,shortUrl&key=KEY&token=TOKEN"
```

## Studio Workflow Summaries

When asked for a project/workflow summary, fetch all open cards from Studio workflow board and group by list. Resolve list names from `references/ofs-boards.md`. Present as a table grouped by stage, noting cards with no due date as a risk flag.

## Output Format

- Use tables for card lists (columns: Job, Stage, Due, URL)
- Flag cards with no due date: ⚠️
- Flag overdue cards: 🔴
- Use short job codes (e.g. CAHL8074) as primary identifier
- For studio summaries, exclude Archive list unless specifically asked
