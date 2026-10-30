# Standards Search API

Ranked search over a standard's identifier, title, and short title. This is the same lookup as the in-app database search and the spotlight search once a keyword is entered. AI-created standards are omitted. Archived standards are omitted. Each account only sees standards it is allowed to see: public standards in sources the account may access, plus standards owned by the account or its company.

An empty keyword does not list the account's kataster. That list is only shown in the in-app spotlight.

## Endpoint

- **Path**: `/research/api/standards-search/`
- **Method**: `GET`
- **View**: `StandardsSearchAPIView`
- **Permission**: authenticated user with `users.api_access` (`IsAPIUser`)
- **OpenAPI**: [standards-search-api.openapi.yaml](./standards-search-api.openapi.yaml)

## Authentication

Send `POST /api/token/` with the account email and password. The response contains a refresh token and an access token. Send the access token on every request:

```http
Authorization: Bearer <access_token>
```

The access token is valid for 30 days. The refresh token is valid for 7 days. Fetch a new access token when you start a new integration run, or exchange the refresh token at `POST /api/token/refresh/`.

A valid login without `users.api_access` returns HTTP 403.

```bash
curl -X POST "https://app.normify.me/api/token/" \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","password":"your-password"}'
```

Use the returned `access` value as the bearer token.

## Query parameters

- `keyword` (required): text to match against identifier, title (`name`), and short title (`name_short`). Surrounding whitespace is removed. Matching is case-insensitive. Punctuation other than `/` is treated as a separator, so `DIN-EN 9001` and `din en 9001` search the same tokens.
- `limit` (optional): how many results to return, best match first. An omitted or blank value uses `20`. Must be an integer from `1` to `100`.

## Usage limits

Each call loads every match and ranks it in memory before `limit` is applied. A short or very common keyword is expensive even when you ask for few rows. Follow the same limits as the in-app spotlight.

| Limit | Value | What to do |
| --- | --- | --- |
| Minimum keyword length | 3 characters after trimming | Do not send a request for 1 or 2 characters. The server rejects those with HTTP 400 and does not search. The spotlight uses the same minimum. |
| Catalog prefix alone | `API`, `CSA`, `DIN`, `IEC`, `ISO`, `JIS`, `SAE`, `UNE`, `VDE`, `VDI` | A keyword that is only one of these is rejected with HTTP 400 and is not searched. `ISO 9001` and `DIN EN 13260` are searched. |
| Maximum keyword length | 200 characters after trimming | Longer values are rejected with HTTP 400. |
| Debounce | 200 ms after the last change | Wait until typing pauses. The spotlight uses `delay:200ms`. Do not send one request per keystroke. |
| In-flight requests | 1 | When the keyword changes, drop the previous request. The spotlight replaces the in-flight call instead of letting them pile up. |
| Result `limit` | 1–100, default 20 | Ask only for the rows you show. `100` is the hard maximum, the same cap as the in-app search. |
| Pace | at most one request every 200 ms | Do not poll this endpoint and do not run it in a tight loop or in parallel from one account. |
| What this is for | interactive lookup | To read a whole source, use [Fetch-Legal-Data-API](https://github.com/Normify-me/legal-data-fetch-api) |

A keyword that is long enough but has no searchable tokens (for example `...`) returns HTTP 200 with an empty `standards` list. It does not run a broad scan.

## Example request

```bash
curl -G "https://app.normify.me/research/api/standards-search/" \
  -H "Authorization: Bearer <token>" \
  --data-urlencode "keyword=ISO 9001" \
  --data-urlencode "limit=20"
```

`--data-urlencode` encodes spaces and slashes. A keyword such as `2006/42` must be encoded (`2006%2F42`) if you build the URL yourself.

## Success response (`200`)

Results are ordered by relevance. Identifier matches come before title and short-title matches. `count` is the number of rows in this response, not the number of matches in the database.

```json
{
  "success": true,
  "data": {
    "keyword": "ISO 9001",
    "limit": 20,
    "count": 1,
    "standards": [
      {
        "normify_identifier": "AbCdEf123456",
        "identifier": "DIN EN ISO 9001",
        "title": "Qualitätsmanagementsysteme",
        "short_title": "ISO 9001",
        "source_name": "DIN"
      }
    ]
  }
}
```

No matches:

```json
{
  "success": true,
  "data": {
    "keyword": "no-such-standard",
    "limit": 20,
    "count": 0,
    "standards": []
  }
}
```

`identifier` is null when the standard has no catalog identifier. `short_title` is an empty string when unset.

## Error response (`400`)

Missing or blank `keyword`:

```json
{
  "success": false,
  "error": {
    "code": "MISSING_KEYWORD",
    "message": "keyword query parameter is required",
    "details": {}
  }
}
```

`keyword` shorter than 3 characters after trimming:

```json
{
  "success": false,
  "error": {
    "code": "KEYWORD_TOO_SHORT",
    "message": "keyword must be at least 3 characters",
    "details": {
      "min_length": 3
    }
  }
}
```

`keyword` is only a catalog prefix such as `DIN` or `ISO`:

```json
{
  "success": false,
  "error": {
    "code": "KEYWORD_TOO_BROAD",
    "message": "keyword matches too many standards; include a number or more words, for example ISO 9001",
    "details": {}
  }
}
```

`keyword` longer than 200 characters after trimming:

```json
{
  "success": false,
  "error": {
    "code": "KEYWORD_TOO_LONG",
    "message": "keyword must be at most 200 characters",
    "details": {
      "max_length": 200
    }
  }
}
```

`limit` is not an integer from 1 to 100:

```json
{
  "success": false,
  "error": {
    "code": "INVALID_LIMIT",
    "message": "limit must be an integer from 1 to 100",
    "details": {
      "max_limit": 100
    }
  }
}
```

## Other errors

- `401`: the bearer token is missing or invalid.
- `403`: the account is signed in but does not have `users.api_access`.
