# Supertext Order API

Documentation for the **Supertext Order API** — the HTTP API used to place and
manage **professional (human) translation orders** with Supertext, upload the
content to translate, receive a completion callback, and download the finished
translation.

> The Order API uses HTTP Basic authentication and lives under `/api/v1` and
> `/api/v1.1`. A newer OAuth2-based API exists for some operations; where both are
> available this document describes the Basic-auth flow, which is what most
> integrations use.

📖 **Browse the rendered reference:**
**<https://supertext.github.io/supertext-order-api-documentation/>** — an interactive,
three-pane API reference generated from `openapi.json`.

**In this repository**

| File | What it is |
|------|------------|
| `README.md` | This guide. |
| `openapi.json` | Machine-readable OpenAPI 3.0 specification (import into Swagger UI, Postman, code generators, etc.). It also drives the [rendered reference site](https://supertext.github.io/supertext-order-api-documentation/). |
| `Supertext Order API 2026.postman_collection.json` | Ready-to-run Postman collection (credentials removed — see [Authentication](#authentication)). |
| `app/` | Next.js source for the [rendered reference site](https://supertext.github.io/supertext-order-api-documentation/) (auto-deployed to GitHub Pages on push to `main`). |

---

## Table of contents

- [Concepts](#concepts)
- [Base URLs & environments](#base-urls--environments)
- [Authentication](#authentication)
- [The typical order flow](#the-typical-order-flow)
- [Endpoints](#endpoints)
  - [Account check](#account-check)
  - [Upload a file](#upload-a-file)
  - [Get a quote](#get-a-quote)
  - [Create an order](#create-an-order)
  - [Get an order](#get-an-order)
  - [Update an order](#update-an-order)
  - [List files for an order](#list-files-for-an-order)
  - [Download a file](#download-a-file)
  - [Provide feedback / reject an order](#provide-feedback--reject-an-order)
  - [List the current person's orders](#list-the-current-persons-orders)
  - [Change an order's status](#change-an-orders-status)
- [The completion callback (webhook)](#the-completion-callback-webhook)
  - [Testing the callback (Write Back Test tool)](#testing-the-callback-write-back-test-tool)
  - [Polling instead of a callback](#polling-instead-of-a-callback)
- [Providing content: Groups vs Files](#providing-content-groups-vs-files)
- [Reference values](#reference-values)
- [Errors](#errors)
- [A complete worked example](#a-complete-worked-example)

---

## Concepts

- **Order** — a professional translation job. One request to *create an order* with
  several target languages produces **one order object per target language**, each
  with its own numeric `Id` (the **OrderId**).
- **Document / File** — the content to translate. You either send the content
  **inline** in the order body (`Groups`), or **upload a file first** and reference
  its `Id` (`Files`). Uploaded and produced files each have a numeric id
  (**DocumentId** / **FileId**).
- **Callback** — when an order is completed, Supertext performs an HTTP `POST` to the
  `CallbackUrl` you supplied, with the order object(s). The finished file(s) are
  listed with `DocumentType: "Final"` and can then be downloaded.
- **ReferenceData** — an opaque string you attach to an order. Supertext echoes it
  back unchanged in the callback, so you can correlate the callback with your own
  system (ticket id, post id, a signed token, …).

---

## Base URLs & environments

All environments live under `supertext.com`:

| Environment | Base URL |
|-------------|----------|
| **Production (live)** | `https://www.supertext.com/` |
| **Staging** | `https://staging.supertext.com/` |
| **Testing** | `https://testing.supertext.com/` |

Use **Staging** or **Testing** for integration work — orders there are not billed
and not actually translated by humans.

> **Historical note:** older examples sometimes use the `supertext.ch` domain
> (e.g. `staging.supertext.ch`). Prefer the `supertext.com` hosts above.

All endpoint paths in this document are **relative to the environment base URL**.

---

## Authentication

The Order API uses **HTTP Basic authentication**:

- **username** = your Supertext **account email**
- **password** = your **Order API Key** (found in your Supertext account settings)

Send it on every request as an `Authorization` header:

```
Authorization: Basic BASE64( "email:orderApiKey" )
```

For example, `jane@example.com:my-api-key` becomes:

```
Authorization: Basic amFuZUBleGFtcGxlLmNvbTpteS1hcGkta2V5
```

```bash
curl -u "jane@example.com:my-api-key" \
  https://staging.supertext.com/api/v1/translation/accountcheck
```

> **Never commit real credentials.** The Postman collection in this repo ships with
> empty credentials; set them via the collection variables `email` and `apiKey`
> (Collection → Variables), or per-request. Rotate any key that has ever been shared
> or committed.

---

## The typical order flow

```
1. (optional) GET  /api/v1/translation/accountcheck     → verify credentials
2. (optional) POST /api/v1/files/files                  → upload content, get DocumentId
3. (optional) POST /api/v1/translation/quote            → price/word count preview
4.            POST /api/v1.1/translation/order           → create the order(s)
                          │
                          ▼  (minutes … days later)
5. Supertext POST  {your CallbackUrl}                    → order completed
6.            GET  /storage/file/{FileId}/{Name}         → download the "Final" file(s)
7. (optional) PUT  /api/v1/order/status/{OrderId}/9      → mark as collected
```

Steps 2 and 4 can each carry the content, so a minimal integration is just
**upload → order → (callback) → download**.

---

## Endpoints

Common headers for JSON endpoints:

```
Accept: application/json
Content-Type: application/json; charset=UTF-8
Authorization: Basic …
```

### Account check

Verify that the credentials are valid.

```
GET /api/v1/translation/accountcheck
```

```bash
curl -u "$EMAIL:$KEY" https://staging.supertext.com/api/v1/translation/accountcheck
```

A `2xx` response means the credentials are accepted.

---

### Upload a file

Upload the document to be translated. `multipart/form-data`.

```
POST /api/v1/files/files
Content-Type: multipart/form-data
```

**Form fields**

| Field | Value | Meaning |
|-------|-------|---------|
| `ElementId` | `0` | New element. |
| `ElementTypeId` | `2` | File element. |
| `DocumentTypeId` | `1` | Document type — `1` for the original (source) document to translate, `3` for a reference file (e.g. a screenshot) that gives the translator visual context but is not itself translated. |
| `file` | *(binary)* | The file. Use a real filename and content type (e.g. `content.html`, `text/html`, or `screenshot.png`, `image/png`). |

```bash
curl -u "$EMAIL:$KEY" \
  -F "ElementId=0" \
  -F "ElementTypeId=2" \
  -F "DocumentTypeId=1" \
  -F "file=@content.html;type=text/html" \
  https://staging.supertext.com/api/v1/files/files
```

**Response** — a JSON array of the uploaded document object(s). Take the returned
`Id` — this is the **DocumentId** you reference from the order's `Files` array.

```json
[ { "Id": 3538699, "Name": "content.html", "DocumentType": "Original" } ]
```

---

### Get a quote

Get the **word count**, **price**, and available **delivery options** (with their
prices and delivery dates) for a piece of content, without placing an order. Upload the
content first (see [Upload a file](#upload-a-file)) and reference it via `Files` — exactly
as you would for an order.

```
POST /api/v1/translation/quote
```

**Request** — select the product with `OrderTypeConfigurationId` **and** its
`OrderTypeId` (see [Reference values](#reference-values)):

```jsonc
{
  "OrderTypeConfigurationId": 167,
  "OrderTypeId": 55,
  "SourceLang": "de-CH",
  "TargetLanguages": ["fr-CH", "en-US"],
  "Files": [ { "Id": 380995, "Comment": "The main story" } ]
  // "Currency": "chf"   // optional — defaults to the account currency
}
```

> The content can be sent inline via `Groups` instead of `Files` (same as an order).
> An older form of this call selects the product with `ServiceTypeId` — see
> [`ServiceTypeId`](#servicetypeid).

**Response** — the word count, currency, and one entry in `Options` per requested
product, each listing its `DeliveryOptions` with price + delivery date:

```jsonc
{
  "Currency": "CHF",
  "CurrencySymbol": "CHF",
  "WordCount": 28,
  "Options": [
    {
      "Name": "Übersetzung PREMIUM",
      "OrderTypeConfigurationId": 167,
      "OrderTypeId": 55,
      "ShortDescription": "For demanding content where every word counts",
      "Description": "Specialised translation incl. revision by a second translator…",
      "DeliveryOptions": [
        {
          "DeliveryId": 1,
          "Name": "Express",
          "Price": 90.0,
          "HourlyRate": null,
          "DeliveryDate": "2026-07-16T10:26:36Z",
          "IsDefaultDeliveryOption": false
        }
        // … one per available delivery speed
      ]
    }
  ],
  "AccountValid": true
}
```

| Field | Description |
|-------|-------------|
| `WordCount` | Authoritative source word count for the submitted content. |
| `Currency` / `CurrencySymbol` | Currency of the quoted prices (the account's currency). |
| `Options[]` | One per requested product (`OrderTypeConfigurationId`). |
| `Options[].DeliveryOptions[]` | Available delivery speeds with `Price`, `DeliveryDate`, `DeliveryId`, and `IsDefaultDeliveryOption`. |
| `AccountValid` | `false` when the request was unauthenticated / the account can't order. |

> **Not every language pair offers the same delivery options** — always build your
> delivery choices from `DeliveryOptions` in the quote rather than assuming a fixed set.

---

### Create an order

Place the translation order. **This is the core endpoint.** Note the **`v1.1`** path.

```
POST /api/v1.1/translation/order
```

Optional query parameter: `?suppressConfirmationEmail=true` to suppress the order
confirmation email.

**Body fields**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `OrderTypeConfigurationId` | int | ✅ | The product/quality tier. **Account-specific** — see [Reference values](#reference-values). |
| `DeliveryId` | int | ✅ | Delivery speed. See [Reference values](#reference-values). |
| `SourceLang` | string | ✅ | Source language. 2-letter (`de`) or regional (`de-CH`). |
| `TargetLanguages` | string[] | ✅ | One or more target languages, e.g. `["fr-CH","en-US"]`. Produces one order per language. |
| `OrderName` | string | ✅ | Human-readable order title. |
| `Groups` **or** `Files` | array | ✅ | The content to translate — see [Groups vs Files](#providing-content-groups-vs-files). |
| `ContentType` | string | – | e.g. `"text/html"`. |
| `CallbackUrl` | string | – | URL Supertext `POST`s to on completion. See [callback](#the-completion-callback-webhook). |
| `ReferenceData` | string | – | Opaque correlation string, echoed back in the callback. |
| `Links` | array | – | Reference links that give the translator context (not translated). See [Reference links](#reference-links-links). |
| `AdditionalInformation` | string | – | Free-text instructions for the translator. |
| `Currency` | string | – | e.g. `"chf"`. |
| `Deadline` | string | – | ISO-8601 desired deadline. |
| `Referrer` | string | – | Free-text source of the order. |
| `SystemName`, `SystemVersion`, `ComponentName`, `ComponentVersion` | string | – | Integration identification (recommended). |

```bash
curl -u "$EMAIL:$KEY" \
  -H "Content-Type: application/json; charset=UTF-8" \
  -X POST https://staging.supertext.com/api/v1.1/translation/order \
  -d '{
    "OrderTypeConfigurationId": 6,
    "DeliveryId": 1,
    "OrderName": "Homepage translation",
    "SourceLang": "de",
    "TargetLanguages": ["fr-CH", "en-US"],
    "ContentType": "text/html",
    "CallbackUrl": "https://yourdomain.com/supertext/callback",
    "ReferenceData": "ticket-4711",
    "Files": [ { "Id": 3538699, "Comment": "File to translate" } ]
  }'
```

**Response** — a JSON **array** with one order object per target language. Each
object's `Id` is the **OrderId**; store it to reconcile the callback later.

```json
[
  { "Id": 715113, "TargetLang": "fr-CH", "Status": "New", "ReferenceData": "ticket-4711" },
  { "Id": 715114, "TargetLang": "en-US", "Status": "New", "ReferenceData": "ticket-4711" }
]
```

#### Reference links (`Links`)

Optionally attach one or more reference URLs that give the translator background or
context for the text. These links are **not** translated — they're just supporting
material. Each entry has a `Url` and an optional `Comment`.

```json
"Links": [
  {
    "Url": "http://en.wikipedia.org/wiki/Amalfi_Cathedral",
    "Comment": "Background to the text"
  }
]
```

---

### Get an order

Fetch a single order with its current status and files.

```
GET /api/v1/order/{orderId}
```

```bash
curl -u "$EMAIL:$KEY" https://staging.supertext.com/api/v1/order/715113
```

The response includes the order's `Status` and a `Files` array; completed
translations appear with `DocumentType: "Final"`.

---

### Update an order

Update a field of an order that already exists, with a
[JSON Patch](https://datatracker.ietf.org/doc/html/rfc6902) document.

```
PATCH /api/v1/order/{orderId}
```

> **Access.** With a standard Order API account you can update the **cost centre** of
> your own orders. The other fields below are writable through the same endpoint, but
> need the internal **`Administrator`** role — your account gets **`403 Forbidden`** for
> them, so ask your Supertext contact if you need one of those changed.

**Body** — a JSON **array** of operations. Use `op: "replace"`; `path` is a slash plus
the property name in PascalCase, exactly as
[`GET /api/v1/order/{orderId}`](#get-an-order) returns it.

```json
[
  { "op": "replace", "path": "/CostCenter", "value": "CC-4711" }
]
```

**Fields**

| Path | Type | Who can update it | What it changes |
|------|------|-------------------|-----------------|
| `/CostCenter` | string (≤ 100) | **You** | Your cost centre, carried onto the invoice. |
| `/OrderName` | string (≤ 100) | Supertext | The order title shown in Supertext and on the invoice. |
| `/AdditionalInformation` | string | Supertext | Free-text instructions for the translator. |
| `/Background` | string | Supertext | Background and purpose of the text. |
| `/TargetAudience` | string | Supertext | Who will read the translation. |
| `/Deadline` | string | Supertext | Requested delivery date, ISO-8601 — see [Deadlines](#deadlines) below. |
| `/DeliveryId` | int | Supertext | Delivery speed — see [`DeliveryId`](#deliveryid). |
| `/IndustryId` | int | Supertext | Subject area. Valid ids: `GET /api/v1/util/industries`. |
| `/TextTypeId` | int | Supertext | Text type. Valid ids: `GET /api/v1/util/texttype`. |
| `/TextStyleId` | int | Supertext | Writing style. Ids are product-specific — ask Supertext. |
| `/Links` | array | Supertext | Reference links (`{ "Url", "Comment" }`) — see [Reference links](#reference-links-links). Replaces the **whole** list. |

On your own account, use only `op: "replace"`, `"add"` or `"remove"`. `"move"` and
`"copy"` are rejected with `403`, because their `from` path would reach a field you may
not change.

Only the fields above are written to the order. On your own account anything else is
rejected with `403`; with the `Administrator` role an unlisted property is instead
**accepted and silently ignored** — a `204` does not mean the value was stored. Either
way, `CallbackUrl`, `ReferenceData`, `ContentType`, `Groups` and `Files` cannot be
changed once the order has been created, and the order's status has
[its own endpoint](#change-an-orders-status). The language pair, the product
(`OrderTypeConfigurationId`) and the price are not supported here either — they affect
pricing and the production workflow, so ask Supertext to change those.

#### Deadlines

Both fields in this section need the `Administrator` role.

`/Deadline` takes an ISO-8601 timestamp, e.g. `2026-07-20T10:00:00Z`. A string the
server cannot parse as a date **clears** the deadline rather than failing, so send the
same format you get back from `GET`.

To let Supertext work the deadline out instead of naming one, add the special
`/RecalculateDeadline` operation — the deadline is then recomputed from the order's
delivery option, respecting business hours and the responsible tenant's timezone:

```json
[
  { "op": "replace", "path": "/DeliveryId", "value": 3 },
  { "op": "replace", "path": "/RecalculateDeadline", "value": true }
]
```

`/RecalculateDeadline` is not a field on the order — only the presence of the operation
matters, and it overrides any `/Deadline` sent in the same request.

> **Changing the delivery speed does not reprice the order**, and it does not reschedule
> work that is already in production. Every change is written to the order's history, but
> nobody is notified — for a change to an order that is already being translated, contact
> Supertext.

**Responses**

| Status | Meaning |
|--------|---------|
| `204 No Content` | The patch was applied. |
| `400 Bad Request` | The patch could not be applied — e.g. a price change on an order that has already been billed, or an unknown language code. |
| `401 Unauthorized` | Credentials rejected. |
| `403 Forbidden` | The patch touches a field you may not update, uses `move`/`copy`, or the order is not one of yours. Nothing is changed. |
| `404 Not Found` | No order was updated. |
| `409 Conflict` | The patch tried to change `/Id`. |
| `500` | The order id does not exist, `path` is not a property of the order, or the update failed. |

```bash
curl -u "$EMAIL:$KEY" \
  -H "Content-Type: application/json; charset=UTF-8" \
  -X PATCH https://staging.supertext.com/api/v1/order/715113 \
  -d '[
    { "op": "replace", "path": "/CostCenter", "value": "CC-4711" }
  ]'
```

---

### List files for an order

```
GET /api/v1/files/files/{orderId}
```

Returns the file objects associated with an order.

---

### Download a file

Download a document (e.g. the finished `Final` file named in the order/callback).

```
GET /storage/file/{fileId}/{fileName}
```

```bash
curl -u "$EMAIL:$KEY" -OJ \
  https://staging.supertext.com/storage/file/1964676/translation.html
```

> Some environments/older integrations expose the same download under
> `GET /filedownloads/file/{fileId}/{fileName}`.

---

### Provide feedback / reject an order

Send a rating and/or comments, or reject a delivered order.

```
POST /api/v1/order/feedback
```

```json
{
  "OrderId": 168110,
  "PostEditing": "true",
  "Rating": 4,
  "Comments": [ { "Comment": "Please keep the product name untranslated." } ]
}
```

---

### List the current person's orders

Paged list of the authenticated person's orders.

```
GET /api/v1/customer/orders/current/{pageSize}/{page}/{personId}
```

Optional filter to exclude certain statuses:
`?queryFilter[IgnoreOrderStatusIds][]={statusId}`

```bash
curl -u "$EMAIL:$KEY" \
  "https://staging.supertext.com/api/v1/customer/orders/current/50/1/598?queryFilter[IgnoreOrderStatusIds][]=22"
```

---

### Change an order's status

```
PUT /api/v1/order/status/{orderId}/{statusId}
```

Most commonly used to mark a collected/handled order as **Collected** (`statusId` = `9`):

```bash
curl -u "$EMAIL:$KEY" -X PUT \
  https://staging.supertext.com/api/v1/order/status/673198/9
```

---

## The completion callback (webhook)

If you supply a `CallbackUrl` when creating the order, Supertext performs an HTTP
`POST` to it when the order is completed. The request body is the **order
object(s)** — the same shape returned by *create an order*, now including the
finished files.

Key points:

- The body may be a **single order object or an array** of them — handle both.
- Your `ReferenceData` is included unchanged — use it to authenticate/correlate the
  callback with your system. (A common pattern is to make `ReferenceData` a signed
  token so a public callback endpoint can verify the call is genuine.)
- The finished translation is the file whose `DocumentType` is **`"Final"`**
  (as opposed to `"Original"`). Download it via
  [`GET /storage/file/{fileId}/{fileName}`](#download-a-file).
- Respond `2xx` to acknowledge receipt.

> **Want to test your callback handler?** You can trigger a writeback for one of your
> orders on demand — without waiting for a live order to be translated — with the
> [Write Back Test tool](#testing-the-callback-write-back-test-tool).

**Example callback body**

```json
{
  "Id": 715113,
  "Status": "Completed",
  "ReferenceData": "ticket-4711",
  "SourceLang": "de",
  "TargetLang": "fr-CH",
  "Files": [
    { "Id": 8811, "Name": "content.html", "DocumentType": "Original", "ContentType": "text/html" },
    { "Id": 8812, "Name": "content.html", "DocumentType": "Final",    "ContentType": "text/html" }
  ]
}
```

### Testing the callback (Write Back Test tool)

While building your integration you usually want to trigger the completion callback
**on demand**, instead of waiting for a real order to be translated and delivered.
Supertext provides a **Write Back Test** tool for exactly that:

<https://staging.supertext.com/services/tools/WriteBackTest>

From there you can fire the completion callback for one of your orders yourself, so you
can verify end-to-end that your endpoint:

1. authenticates / correlates the call via your `ReferenceData`,
2. downloads the `Final` file, and
3. writes the translation back into your system correctly.

**Access.** The tool requires the special **`API Developer`** role. Any Supertext IT
employee can grant it to your account — just ask your Supertext contact.

> This is a staging tool for integration testing: point your order's `CallbackUrl` at
> the endpoint you want to exercise, then use the tool to (re)send the writeback as
> often as you need while you build and debug.

### Polling instead of a callback

If you can't expose a public callback endpoint — i.e. writeback isn't possible in your
environment — you can **poll for completion** instead:

1. Create the order with **`ReferenceData` and `CallbackUrl` left empty.** When you
   poll you need neither a webhook URL nor a signed correlation token, so it's cleaner
   to omit both.
2. Poll [`GET /api/v1/order/{orderId}`](#get-an-order) until the order's status reaches
   **Delivered (`8`)**, then download the `Final` (`2`) file(s).
3. **Important — mark the order as collected yourself.** After downloading the finished
   files, set the status to **Collected (`9`)** with
   [`PUT /api/v1/order/status/{orderId}/9`](#change-an-orders-status). When polling,
   Supertext does not advance the order for you, so if you skip this the order stays
   delivered-but-uncollected.

---

## Providing content: Groups vs Files

You can send the content to translate in **two ways**. Use exactly one per order.

### `Files` — reference an uploaded document

Upload with [`POST /api/v1/files/files`](#upload-a-file), then reference the id:

```json
"Files": [ { "Id": 3538699, "Comment": "Main story" } ]
```

### `Groups` — inline content

Structured content embedded directly in the order body. Each **group** has an id and
a list of **items**; each item has an id and its `Content`. The same ids come back on
the translated side so you can map translations onto your original structure.

```json
"Groups": [
  {
    "Context": "Some Node",
    "GroupId": "Group1",
    "Items": [
      { "Id": "1", "Content": "<div>This is the content of group 1</div>", "Comment": null, "Context": null },
      { "Id": "2", "Content": "This is <i>new</i> content of group 1",     "Comment": null, "Context": null }
    ]
  }
]
```

| Field | Level | Description |
|-------|-------|-------------|
| `GroupId` | group | Your identifier for the group. |
| `Context` | group / item | Optional context shown to the translator. |
| `Items[].Id` | item | Your identifier for the string (returned unchanged). |
| `Items[].Content` | item | The text/HTML to translate. |
| `Items[].Comment` | item | Optional per-item comment. |

---

## Reference values

> ⚠️ **Product ids are account-specific.** `OrderTypeConfigurationId` (and the older
> `ServiceTypeId`) values depend on the products configured for your account. The
> values below are examples/known values — confirm yours with Supertext or via a
> quote. `DeliveryId` and the upload constants are stable.

### `DeliveryId`

| Id | Delivery |
|----|----------|
| 1 | Express |
| 2 | 24 hours |
| 3 | 48 hours |
| 4 | 3 days |
| 5 | 1 week |

### `OrderTypeConfigurationId` / `OrderTypeId` (example values)

Each product is identified by an `OrderTypeConfigurationId` (used to create an order) and
a matching `OrderTypeId` (used, together with it, to request a [quote](#get-a-quote)). Both
are returned in each quote's `Options[]`.

| OrderTypeConfigurationId | OrderTypeId | Product |
|----|----|---------|
| 166 | 54 | Translation BASIC |
| 167 | 55 | Translation PREMIUM |
| 168 | 56 | Translation CREATIVE |

*(Other accounts may use different ids — e.g. `6` appears in the sample requests.)*

### `ServiceTypeId`

An older service-type selector used by the **[quote](#get-a-quote)** endpoint
(`POST /api/v1/translation/quote`). Like `OrderTypeConfigurationId`, its values are
**account-specific**. When creating an order you use `OrderTypeConfigurationId` instead —
`ServiceTypeId` is not required on the order endpoint.

| Id | Service |
|----|---------|
| 46 | Standard |

> These ids can be **client-specific** — confirm the `ServiceTypeId` values available to
> your account with Supertext.

### Upload constants (`POST /api/v1/files/files`)

| Field | Value | Meaning |
|-------|-------|---------|
| `ElementId` | `0` | New element |
| `ElementTypeId` | `2` | File |
| `DocumentTypeId` | `1` | Document type — `1` = original document to translate, `3` = reference file (e.g. screenshot) |

### `DocumentTypes`

| Id | Value | Meaning |
|----|-------|---------|
| 1 | `Original` | The source document you supplied. |
| 2 | `Final` | The finished translation to download. |
| 3 | `Reference` | Supporting material (e.g. a screenshot) attached for the translator's context; not translated. |

### Order status (partial)

Order objects carry a textual `Status` (e.g. `New`, `In progress`, `Quality check`),
backed by a numeric status id.

| Status id | Meaning |
|-----------|---------|
| 8 | **Delivered** — the order is finished. **Poll for this** when waiting for a completed order. |
| 9 | Collected |

> **Waiting for a finished order:** either supply a `CallbackUrl` (Supertext POSTs the
> completed order to it — see [the callback](#the-completion-callback-webhook)), or
> [poll](#polling-instead-of-a-callback) [`GET /api/v1/order/{orderId}`](#get-an-order)
> until the status reaches **Delivered (8)**, then download the `Final` (2) file(s). When
> polling, remember to mark the order **Collected (9)** yourself afterwards.

---

## Errors

- Authentication problems return **`401 Unauthorized`**.
- Validation problems return a **`4xx`** with a JSON or text body describing the
  issue.
- Always check the HTTP status code; a non-`2xx` response means the operation did
  not succeed, and the body usually carries a human-readable message.

---

## A complete worked example

```bash
BASE=https://staging.supertext.com
AUTH=(-u "jane@example.com:my-api-key")

# 1. Verify credentials
curl "${AUTH[@]}" "$BASE/api/v1/translation/accountcheck"

# 2. Upload the content, capture the DocumentId
DOC_ID=$(curl -s "${AUTH[@]}" \
  -F "ElementId=0" -F "ElementTypeId=2" -F "DocumentTypeId=1" \
  -F "file=@content.html;type=text/html" \
  "$BASE/api/v1/files/files" | python -c 'import sys,json;print(max(d["Id"] for d in json.load(sys.stdin)))')

# 3. Create the order(s)
curl "${AUTH[@]}" -H "Content-Type: application/json; charset=UTF-8" \
  -X POST "$BASE/api/v1.1/translation/order" \
  -d "{
    \"OrderTypeConfigurationId\": 6,
    \"DeliveryId\": 1,
    \"OrderName\": \"Homepage\",
    \"SourceLang\": \"de\",
    \"TargetLanguages\": [\"fr-CH\"],
    \"ContentType\": \"text/html\",
    \"CallbackUrl\": \"https://yourdomain.com/supertext/callback\",
    \"ReferenceData\": \"ticket-4711\",
    \"Files\": [ { \"Id\": $DOC_ID, \"Comment\": \"Main story\" } ]
  }"

# 4. …later, your CallbackUrl receives the completed order.
#    Download the "Final" file:
curl "${AUTH[@]}" -OJ "$BASE/storage/file/8812/content.html"
```
