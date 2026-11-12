<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/rawchunks/app/main/public/branding/logo.svg">
  <img alt="Rawchunks" src="https://raw.githubusercontent.com/rawchunks/app/main/public/branding/logo-light.svg" width="360">
</picture>

## What Rawchunks Is

Rawchunks ([rawchunks.space](https://rawchunks.space) | [app.rawchunks.space](https://app.rawchunks.space) | [docs.rawchunks.space](https://docs.rawchunks.space)) is an open-source music studio, browser DAW, and developer API built from scratch in 6 weeks during the Nebius x NVIDIA Global AI Hackathon.

Closed AI music products treat music generation as a slot machine. You type a prompt, get a baked stereo file, and if bar 9 has the wrong vocal phrasing or you want a tighter bassline in the drop, you throw away the track and roll the dice again. You cannot surgically rework a 4-bar section against isolated stems, program MIDI or live-coded synths on the same timeline, train and reuse portable voice adapters, call an open API without enterprise lock-in, or own and license the resulting catalog onchain.

Rawchunks replaces the one-shot prompt box with a full studio architecture:

1. **Multimodal Songwriter & Algorithmic Composer Agents on Nebius Token Factory** powered by **`nvidia/Nemotron-3-Ultra-550b-a55b`** and **`nvidia/Nemotron-3.5-Super-VL`**.
2. **High-Throughput Audio Diffusion & Stem Engine on NVIDIA L4 GPUs** running custom-compiled **`ACE-Step 1.5 XL`** (`Turbo`, `SFT`, and `Base` checkpoints), **`acestep-5Hz-lm-0.6B`** on vLLM, **`Qwen3-Embedding-0.6B`**, **`htdemucs`**, and on-demand **Rank-16 Voice LoRA training**.
3. **Zero-GPU Browser DAW** with multitrack stem lanes, range-based AI editing (Rework, Layer, Extract, Separate, Extend, Restyle, Accompany), a piano roll, 16-step drum sequencer, Strudel live-coding REPL, sampler, 3D spatial panner, and 8-stage FX rack.
4. **Open Studio API & Programmable Licensing** supporting both standard API keys and accountless HTTP 402 (`x402`) USDC micropayments alongside onchain song and remix licensing on Solana and Base.

---

## How Judge Testing Works (100 Free Credits in 2 Clicks)

You can test the full production stack on [app.rawchunks.space](https://app.rawchunks.space) immediately:

1. Sign in at [app.rawchunks.space](https://app.rawchunks.space).
2. Click **`SESSION`** in the top bar, open **`CAMPAIGN`**, and claim **100 free Credits** (`$1.00` USD equivalent, enough for 20 Fast song generations or dozens of surgical stem/range edits).
3. Test the **Songwriter Agent (`VIBE`)** in Studio with a text prompt, an image, or a microphone hum, then open the track in the **Editor** to test range edits and the **Algorithmic Composer (`LIVE CODE -> AI CODE`)**.

---

# PayPal and AG Grid integration

## Current packages

| Package | Current locked version | Application use |
| --- | --- | --- |
| `@paypal/react-paypal-js` | `10.6.1` | SDK provider and script-loading state. |
| `@paypal/paypal-js` | `11.3.0` | SDK loader dependency of `@paypal/react-paypal-js`. |
| `@paypal/sdk-constants` | `1.0.158` | SDK constants dependency of `@paypal/react-paypal-js`. |
| `ag-grid-react` | `36.3.0` | React table component. |
| `ag-grid-community` | `36.3.0` | Dependency of the Grid packages; app imports module registration, column definitions and renderer types. |
| `ag-grid-enterprise` | `36.3.0` | Installed package; current tables register Community modules. |

Versions: [app/package-lock.json](https://github.com/rawchunks/app/blob/main/package-lock.json). Backend PayPal calls use `fetch`; `PayPal` accepts an injected HTTP transport.

## PayPal

### SDK loading and hosted fields

`SessionProvider` wraps its children in `PayPalScriptProvider`. It chooses `VITE_PAYPAL_CLIENT_ID`, then the balance response's `paypalClientId`, then an empty string. Provider options are `components: "card-fields"`, `currency: "USD"`, `intent: "capture"`, and `clientId: paypalClientId || "test"`. The provider key is the client ID or `"pending"`; `deferLoading: !paypalClientId` delays loading until configuration arrives.

`FundingPanel` mounts its card panel while hidden so the hosted-field iframes can initialize before the user selects Card. Mounting doesn't create an order; PayPal calls the supplied `createOrder` callback during submission.

These are all PayPal frontend API calls used by the application:

| Call | Exact use |
| --- | --- |
| `PayPalScriptProvider` | Loads/configures the SDK for descendants. |
| `usePayPalScriptReducer()` | Reads `isPending`, `isResolved`, `isRejected`; no reducer dispatch here. |
| `window.paypal.CardFields(config)` | Creates one hosted-card form after the script resolves. |
| `instance.isEligible()` | Checks merchant eligibility before rendering fields. |
| `instance.NameField(options)` | Cardholder name; placeholder `CARDHOLDER NAME`. |
| `instance.NumberField(options)` | Card number; placeholder `0000 0000 0000 0000`. |
| `instance.ExpiryField(options)` | Expiry; placeholder `MM / YY`. |
| `instance.CVVField(options)` | Security code; placeholder `CVV`. |
| `field.render(host)` | Renders each field into a fresh DOM host; the four promises run together. |
| `form.getState()` | Reads `isFormValid` before submission; a rejected state read becomes `null` and submission continues. |
| `form.submit()` | Starts card submission after field-render promises finish. |
| `field.close()` | Closes each iframe on effect cleanup; cleanup also empties the host slots. |

Every field receives `styleOptions: { disablePrerender: true }`. Rawchunks passes iframe styles through `readPayPalCardStyle()` because the fields live outside the app's normal DOM styling.

### Frontend function inventory

In [connector.tsx](https://github.com/rawchunks/app/blob/main/src/components/connector.tsx#L178):

| Function or callback | Responsibility |
| --- | --- |
| `formatPayPalError(error, fallback)` | Extracts an `Error.message`, string or object's string `message`; otherwise uses the fallback. |
| `readPayPalCardStyle(element)` | Reads `PAYPAL_STYLE_VARS` with `getComputedStyle`; copies nonempty values into PayPal selector/property maps. |
| `PayPalCardFields(props)` | Owns SDK readiness, DOM slots, form/render refs, errors, submit state and field cleanup. |
| `inputEvents.onChange` | Clears the displayed error. |
| `CardFields.createOrder` | Uses `callbacksRef` for the current parent callback; `orderInFlightRef` shares one in-flight promise and resets it in `finally`. |
| `CardFields.onApprove` | Awaits the parent's approval callback; converts failures into the error callback. Callback data includes `orderID` and optional `liabilityShift`; application code doesn't branch on `liabilityShift`. |
| `CardFields.onError` | Formats and forwards the SDK error. |
| `handleSubmit()` | Blocks duplicate/disabled submits, clears errors, waits for rendering, checks form validity, calls `submit`, resets both submit guards in `finally`. |
| `CardCheckout(props)` | Selects a pack; keeps current order parameters in a ref and an operation/request key across the checkout. |
| `CardCheckout` version effect | Clears operation/error and creates a new UUID when `version` changes. |
| Pack button `onClick` | Changes the selected pack and resets operation/key/error; pack buttons disable while an operation exists. |
| `CardCheckout.createOrder` | Posts the current pack, balance intent, network and wallet; saves order/operation IDs and the operation recovery key. On failure, clears operation and rotates the UUID. |
| `CardCheckout.onApprove` | Requires the approved ID to match the saved order, captures it, awaits `completeCreditPurchase`, clears order/key and calls `onComplete`. Errors also clear/rotate the local operation/key. |
| `CardCheckout.onError` | Clears an existing operation and rotates its UUID when an error arrives; updates the error strip. |
| Login button `onClick` | Cancels the panel and opens the login → terms → funding gate with Card selected. |

`callbacksRef` updates on every render, while the field initialization effect depends on script resolution. This lets callbacks read current packs/account context without rebuilding all four iframes.

PayPal loading and checkout entry points in [session.tsx](https://github.com/rawchunks/app/blob/main/src/components/session.tsx#L530):

| Function/callback | PayPal use |
| --- | --- |
| `SessionProvider` / `paypalOptions` | Selects client ID and supplies the SDK options above. |
| `FundingPanel` | Pre-mounts `CardCheckout` in the hidden Card panel to initialize hosted fields. |
| `handleCompleteCreditPurchase` | Awaits purchase confirmation, then selects the card balance. |
| `Session` checkout-event `handler` | Opens the requested card checkout and increments its version. |
| `handleCloseModal`, `handleDone` | Clear and close the checkout panel. `CardCheckout` calls `handleDone` after confirmed payment. |

`PayPalCardFieldsProps` supplies `createOrder`, `onApprove`, `onError`, `onCancel`, `cancelLabel`, `submitLabel`, `submitVariant`, `disabled`, `error`, `notice`, `showBrand`. Cancellation is an app button callback. `onApprove` receives `{ orderID, liabilityShift? }`.

### Pack checkout

`CardCheckout.createOrder` posts the selected pack, `{ kind: "balance" }`, account and network. It stores `{ operationId, orderId }`; approval must return that same order ID before capture. `CheckoutIntent` is `{ kind: 'balance' }`; each `CreditPack` contains string `id`, `amountAtomics` and `usd` fields.

| `packId` | PayPal USD amount | Atomic amount | Credits |
| --- | --- | --- | --- |
| `usd-10` | `10.00` | `10000000` | 1,000 |
| `usd-25` | `25.00` | `25000000` | 2,500 |
| `usd-100` | `100.00` | `100000000` | 10,000 |

One USD uses 1,000,000 atomic units; one Credit uses 10,000. Backend `packs` supplies the catalog.

| Checkout helper | Function |
| --- | --- |
| `promptCardCheckout(intent)` | Dispatches `rawchunks:card-checkout`; current intent is `{ kind: "balance" }`. |
| `creditsApi(path, body?, idempotencyKey?, options?)` | Calls `/api/credits/{path}`: GET without body, POST with JSON body. Adds supplied idempotency/account/token headers and raises response errors. |
| `purchaseCacheKey(owner)` | `rawchunks:credit-purchase:{lowercase owner}` stores the pending operation ID. |
| `compact`, `formatCredits`, `formatDollars`, `creditLabel` | Format checkout pack/balance amounts; `creditLabel` combines Credits and USD. |
| `useCreditAccount` / `refresh` | Loads public checkout configuration without an account, or account balance/configuration when connected. Refreshes connected balances every 15 seconds, on window focus and after capture; ignores responses for a changed account. |
| `completePurchase(operationId)` | Reads the operation, requires `credited`, refreshes the balance and removes the pending key. |
| Purchase `recover` callback | Reads that saved operation immediately and every 10 seconds until confirmation succeeds. |
| `readCachedPayPalClientId`, `writeCachedPayPalClientId` | Persist/read `paypalClientId`, environment and packs in `rawchunks:paypal-config`. |
| `readCachedCredits`, `writeCachedCredits` | Preserve the displayed purchased balance and checkout configuration across refreshes. |

Sources: [credits.ts](https://github.com/rawchunks/app/blob/main/src/lib/credits.ts), [use-credits.ts](https://github.com/rawchunks/app/blob/main/src/hooks/use-credits.ts#L168), [cache.ts](https://github.com/rawchunks/app/blob/main/src/lib/cache.ts#L126).

### License checkout

[Licenses](https://github.com/rawchunks/app/blob/main/src/components/songs/licenses.tsx) mounts `PayPalCardFields` when `cardMode && testnetMode`.

| Function/callback | PayPal use |
| --- | --- |
| `Licenses` | Owns the license card modal, pending order and request key. |
| `openModal` | Clears card/order/error state and creates a request UUID. |
| `closeModal` | Closes the modal and clears card mode/error. |
| `createFiatOrder(idempotencyKey)` | Checks the current license price/availability and posts song ID, chain song ID, network, wallet and atomic price to the order endpoint. |
| Hosted-fields `createOrder` | Saves operation/order IDs and price; returns the PayPal order ID. |
| Hosted-fields `onApprove` | Checks saved order ID, awaits `completeFiatOrder`, clears the pending operation and closes the modal. |
| Hosted-fields `onError` | Shows the error; a nonempty error clears an existing operation and resets its request key. |
| Hosted-fields `onCancel` | Returns from card entry by setting `cardMode` to `false`. |
| `completeFiatOrder` | Guards concurrent completion, captures the saved order, accepts `completed`/`credited`, rechecks ownership and continues the license purchase if needed; resets pending state in `finally`. |

Source for the two order helpers: [use-license.ts](https://github.com/rawchunks/app/blob/main/src/hooks/use-license.ts#L278).

### Application endpoints used by PayPal checkout

Account requests carry `x-session-user-address`; the checkout helper adds `x-wallet-auth-token` when available. Create-order requests require `x-idempotency-key` of 8–128 characters.

| Method and path | Request | Response |
| --- | --- | --- |
| `GET /api/credits/promo` | Public configuration read. | `{ paypalClientId, packs, environment, promo }`; checkout uses client ID/packs. |
| `GET /api/credits/balance` | Account header. | Account balance plus `paypalClientId`, `packs`, `environment`; checkout displays available funds. |
| `POST /api/credits/paypal/create-order` | `packId`; optional `checkout` object with optional string `kind` and passthrough fields, `network`, `walletAddress`. | `{ operationId, orderId, status }`. |
| `POST /api/credits/paypal/capture-order` | `orderId`, optional `operationId`, optional `card`. | `{ operationId, status, orderId?, captureId?, coinflowPaymentId? }`. |
| `POST /api/credits/license/create-order` | `songId`, `chainSongId`, `network`, `walletAddress`, `amountAtomics`. | `{ operationId, orderId, status }`. |
| `POST /api/credits/license/capture-order` | Same capture schema as pack checkout. | `{ operationId, status, orderId?, captureId?, coinflowPaymentId? }`. |
| `GET /api/credits/operations/{id}` | Account header and operation ID. | `{ id, status, amountAtomics, orderId, kind, coinflowPaymentId }`. |
| `POST /api/credits/paypal/webhook` | PayPal event and signature headers. | `{ received: true }`. |
| `POST /api/payments/paypal/webhook` | Same webhook contract. | Alias to the same handler. |

IDs allow 1–128 characters. License `amountAtomics` matches `^[1-9][0-9]{0,17}$`. Optional `card` contains `number` (12–24 characters), `expiry` (4–10), `securityCode` (3–4), optional `name` (≤128). Hosted fields send order IDs; the alternate card body maps to `payment_source.card` on capture, renaming `securityCode` to `security_code` and defaulting name to `License Buyer`.

`creditsRouter` registers these handlers under `/api` when `CREDITS_ENABLED === "true"`; its `body` helper validates requests and nested `key` validates the idempotency header. Pack network uses the body, then `x-network-id`; wallet defaults to the selected account. Operation reads attempt `PayPal.reconcile` for pending orders before returning current state.

Sources: [router.ts](https://github.com/rawchunks/backend/blob/main/credits/router.ts#L43), [registration](https://github.com/rawchunks/backend/blob/main/index.ts#L108).

### PayPal REST calls

`PayPal.base` chooses `https://api-m.paypal.com` for `live` and `https://api-m.sandbox.paypal.com` for `sandbox`.

| Method/path on that host | Payload / handling |
| --- | --- |
| `POST /v1/oauth2/token` | Basic client ID/secret; form body `grant_type=client_credentials`; 15-second timeout. Cache token until `expires_in - 60` seconds. |
| `POST /v2/checkout/orders` | `intent: "CAPTURE"`; one purchase unit with `custom_id`, environment-prefixed `invoice_id`, USD amount and merchant ID or email. |
| `GET /v2/checkout/orders/{orderId}` | Reads the authoritative order before validation/capture and during recovery. |
| `POST /v2/checkout/orders/{orderId}/capture` | `{}` for hosted fields; optional `payment_source.card` for the alternate raw-card body. |
| `POST /v1/notifications/verify-webhook-signature` | Header-derived `auth_algo`, `cert_url`, `transmission_id`, `transmission_sig`, `transmission_time`, configured `webhook_id` and `webhook_event`. |

`request()` sets bearer authorization, JSON and `Prefer: return=representation`; non-OAuth calls use a 20-second timeout. Both OAuth and API requests reject redirects. Order/capture requests set `PayPal-Request-Id: {environment}-{id}`. Failed responses expose the first provider issue and HTTP status.

### Every PayPal backend function

Source: [paypal.ts](https://github.com/rawchunks/backend/blob/main/credits/paypal.ts).

| Function/method | Exact responsibility |
| --- | --- |
| `packs` | Exported readonly pack catalog above. |
| `fail` | Builds an Error with `statusCode`; default 502. |
| `blockchainForNetwork` | Maps `eip155:84532`/`8453` to `base`, `eip155:42431`/`4217` to `tempo`, `solana:*` to `solana`; returns `null` otherwise. |
| `PayPal.constructor(store, transport = fetch)` | Holds the environment-scoped store and HTTP transport; token cache belongs to the instance. |
| `base` getter | Selects the PayPal host. |
| `request(path, method, body, id)` | Enforces allowed paths, authenticates/caches OAuth, sends JSON, returns parsed response or `{}` for 204. Provider errors include first issue and `providerStatus`. |
| `operation(id, owner?)` | Reads an environment-scoped operation; returns 404 for absence or owner mismatch. |
| `expectedUsd(op)` | Resolves a purchase by pack atomic amount, or reads a license's stored `context.usd`. |
| `create(owner, packId, key, options)` | Stores account/purchase operation and pack/network/wallet context; deterministic ID is the first 36 hex characters of SHA-256 over environment/owner/key. Replay compares pack amount. |
| `createLicense(owner, input, key)` | Maps network, stores song/wallet/price context; rounds atomics to cents with a one-cent minimum. Hash includes `license`; replay compares kind, amount, song ID and network. |
| `merchant(op)` | Chooses stored merchant identity, then configured merchant ID/email. Email fallback is `PAYPAL_MERCHANT_EMAIL`, then `PAYPAL_EMAIL`, lowercased. |
| `validateOrder(op, order)` | Requires matching order ID, exactly one purchase unit, matching custom/invoice IDs, USD and exact formatted amount; verifies merchant and saves merchant ID/email plus `orderValidated`. |
| `createOrder(id)` | Rejects completed local operations; rereads an existing order, reconciles approved/completed/card-attached orders and returns 409 for their reuse. Otherwise reuses the saved ID. New orders save `orderRequestedAt`, enforce a six-hour creation-recovery window, create/read/validate the order and persist its ID. |
| `coinflowCheckout(op, captureId, payerEmail?, markCompleted = true)` | Post-capture handoff; returns/stores `coinflowPaymentId`. With `markCompleted`, also marks the license payment `completed` and saves its capture ID. License reconciliation requires this call; pack reconciliation calls it conditionally and still credits the verified PayPal capture if the handoff fails. |
| `capture(owner, id?, orderId, card?)` | Resolves missing operation ID by order/owner, checks ownership/order match, returns terminal local state, validates provider order, captures if needed and reconciles. A failed capture response triggers an authoritative reread before throwing. |
| `reconcile(id)` | Returns terminal local state; creates a missing order; reads/validates order, captures APPROVED or CREATED-with-card orders, verifies exactly one COMPLETED USD capture, then hands the verified capture to license completion or purchased-balance crediting. |
| `webhook(headers, event)` | Verifies through PayPal; requires `verification_status: "SUCCESS"` and string event ID, then inserts the event once into `credit_events`. It doesn't credit the balance directly. |

Local `purchase` operations finish as `credited`; local `license` operations finish as `completed`. Provider statuses use uppercase values. New order creation returns the provider status; reuse returns the stored local status. `reconcile()` may return a provider status while the local operation remains pending.

### PayPal configuration, styling and recovery

| Variable | Use |
| --- | --- |
| `VITE_PAYPAL_CLIENT_ID` | App's preferred public client ID. |
| `PAYPAL_CLIENT_ID`, `PAYPAL_CLIENT_SECRET` | Backend OAuth client credentials; configuration endpoints return the client ID. |
| `PAYPAL_ENV` | Selects `sandbox` or `live` PayPal host and payment-operation namespace. |
| `PAYPAL_MERCHANT_ID` | Expected merchant; takes priority over email. |
| `PAYPAL_MERCHANT_EMAIL`, `PAYPAL_EMAIL` | Expected payee email, in that priority order. |
| `PAYPAL_WEBHOOK_ID` | Webhook signature-verification target. |

`PAYPAL_STYLE_VARS` maps body/input/card-icon/placeholder/hover/focus/valid/invalid/focused-invalid rules to `--paypal-card-*` CSS variables. `readPayPalCardStyle` resolves background, text, borders, shadows, font, padding, height and transitions before creating hosted fields. Source: [components.css](https://github.com/rawchunks/app/blob/main/src/styles/components.css#L3311).

New order requests use `order-{operationId}` and capture uses `capture-{operationId}`, both environment-prefixed in PayPal's request header.

[The recovery timer](https://github.com/rawchunks/backend/blob/main/index.ts#L113) calls `reconcile` every 30 seconds for up to 20 pending purchase operations from the last three days. License order reads/capture requests also reconcile. Verified webhook events enter storage once by event ID; the webhook handler doesn't capture an order itself.

[credits-paypal.test.ts](https://github.com/rawchunks/backend/blob/main/tests/unit/credits-paypal.test.ts) covers order/capture validation, replay, lost capture responses, owner checks, webhook rejection and CREATED-with-card recovery.

## AG Grid

### Registration and API calls

[utils.ts](https://github.com/rawchunks/app/blob/main/src/components/ag/utils.ts) imports `AllCommunityModule` as `GridCommunityModule` and `ModuleRegistry` as `GridModuleRegistry`; it calls `GridModuleRegistry.registerModules([GridCommunityModule])`.

Songs, focused genre details and activity each render `AgGridReact` with React-computed rows and column definitions. Grid updates use those props and stable row IDs. There are no imperative Grid API calls in these components.

| AG Grid property/API | Use |
| --- | --- |
| `rowData` | React-computed sorted songs, genre slices or activity events. |
| `columnDefs: ColDef[]` | Field, width/flex, custom renderer/formatter and header definitions below. |
| `getRowId` | Stable song ID, genre string or event ID. |
| `defaultColDef` | `suppressHeaderMenuButton: true`, `sortable: false`, `resizable: false`. |
| `suppressScrollOnNewData` | Preserves table scroll position on data refresh. |
| `suppressCellFocus` | Disables the grid's cell-focus UI. |
| `animateRows` | Enables row movement animation. |
| `headerHeight`, `rowHeight` | Compact/focused/mobile dimensions below. |
| `onRowClicked` | Songs only: passes the clicked song to `onSelectSong` when focused. |
| `headerComponent`, `headerComponentParams` | Rawchunks header tooltip and alignment. |
| `cellRenderer(ICellRendererParams)` | React song/genre/receipt content. |
| `valueFormatter` | Play-count and duration strings. |
| `field`, `flex`, `minWidth`, `width`, `headerClass`, `cellClass` | Bind data and size/align each column. |

| Grid | Columns | Header / row height (mobile, desktop) |
| --- | --- | --- |
| Songs compact | SONG, PROFIT, PLAYS | 28/30; 38/40. |
| Songs focused desktop | Adds STREAMED, RATIO, GENRE | 30; 40. |
| Songs focused mobile | Adds RATIO | 28; 38. |
| Genres focused | One custom Genre/details column | 0; 36/38. |
| Activity compact | One custom Activity column | 0; 58/60. |
| Activity focused | Same column with expanded receipt | 0; 104/108. |

Song columns: title `flex: 1.6`, min width 105/130; profit width 76/88; plays 58/74; focused desktop streamed 96, ratio 72, genre 196; focused mobile ratio 64. Genre/activity columns use `flex: 1`. Numeric columns use `cm-ag-right` for header/cell alignment.

### Table functions and callbacks

| Source | Functions and use |
| --- | --- |
| [index.tsx](https://github.com/rawchunks/app/blob/main/src/components/ag/index.tsx) | `Earn` owns table focus and supplies rows. Media-query `onChange` updates `isMobile`; `toggleFocus` expands/collapses a block. `networkOptions`/`viewSwitchOptions` define controls. `normalizedSongs` builds table rows; `unifiedEvents` builds feed rows using `windowCutoffMs`. |
| [songs.tsx](https://github.com/rawchunks/app/blob/main/src/components/ag/songs.tsx) | `SongsBlock` owns song/genre view, sort, search and genre metric. `sortedSongs` filters/sorts input. `handleSongRowClicked` requires focus, row song and `onSelectSong`. `songsColumnDefs` constructs responsive columns. |
| [genres.tsx](https://github.com/rawchunks/app/blob/main/src/components/ag/genres.tsx) | `GenreBadges`/`cleanGenres` split, trim and deduplicate labels. `GenreBreakdown`/`genreSlices` aggregate rows by genre. `genreDetailCols` builds the single details column; its `cellRenderer` displays metric totals. |
| [feed.tsx](https://github.com/rawchunks/app/blob/main/src/components/ag/feed.tsx) | `ActivityFeed` owns feed mode and filters. `filteredPlayEvents`, `settlementEvents`, `streamEventsCount`, `licenseEventsCount`, `totalSettledUsd` select/count/sum rows; `feedRows` selects the active mode's rows. `feedColumnDefs` builds the receipt column and its `cellRenderer`. |
| [dropdown.tsx](https://github.com/rawchunks/app/blob/main/src/components/ag/dropdown.tsx) | `AgHeaderTooltip` renders the grid header tooltip. `BlockDropdown` selects view/sort/filter; `handleOutside` and `handleKey` close it, option `onClick` calls `onChange` and closes. |
| [utils.ts](https://github.com/rawchunks/app/blob/main/src/components/ag/utils.ts#L224) | `getSongRowId`, `getGenreRowId`, `getFeedRowId` return stable IDs. `formatMs` formats listening duration; `shortTx` abbreviates hashes; `timeAgo` formats elapsed time; `ratioTone` selects ratio color. |
| [use-earnings.ts](https://github.com/rawchunks/app/blob/main/src/hooks/use-earnings.ts#L14) | Grid renderers call `formatUsd` for monetary values and `getNetworkLogo` for network icons. `useCreatorEarnings` fetches the input data described below. |

#### Column renderers

| Renderer/formatter | Output |
| --- | --- |
| Song `cellRenderer` | Cover and title with full genre/network tooltip. Missing/erroring image uses `/branding/mark.svg`; image `onError` sets that fallback. |
| Profit `cellRenderer` | Formatted `profitUsd`. |
| Plays `valueFormatter` | Localized `rawPlays` count. |
| Streamed `valueFormatter` | `formatMs(streamedMs)`, focused desktop only. |
| Desktop/mobile ratio `cellRenderer` | `ratioPct` to one decimal; emerald ≥65%, amber ≥35%, danger below. |
| Genre `cellRenderer` | `GenreBadges(allGenres)`, focused desktop only. |
| Genre-details `cellRenderer` | Genre/color marker, song count tooltip, profit, plays, duration and average ratio. |
| Activity `cellRenderer`, plays mode | Song, genre badges, network, time, duration or license royalty amount; expanded rows add streamed/duration ratio and royalties. |
| Activity `cellRenderer`, settlements mode | Song, Settled badge, royalties, network, hash/explorer link; expanded rows add creator share, platform fee and listening duration. |

`AgHeaderTooltipParams` contains `displayName`, `tooltip`, `align`. `BlockDropdownProps` contains `value`, `options`, `onChange`, `ariaLabel`, optional `align`, `chevron`, `triggerIcon`, `variant`. Dropdown clicks and focused table-wrapper clicks stop propagation to the section's focus toggle; explorer-link clicks do the same.

### Every filter/control

| Constant / state | Values and behavior |
| --- | --- |
| `DATE_WINDOW_OPTIONS` / `dateWindow` | `24h`, `7d`, `30d`, `180d` (label LAST 6M). Standalone Earn default 7d; My-Page passes 30d. |
| `SOURCE_OPTIONS` / `sourceFilter` | `all`, `streams`, `licenses`; default all. |
| `networkOptions` / `globalNetworkFilter` | `all`, `base`, `solana`, `tempo`; family filter. |
| `viewSwitchOptions` / `earnView` | `list`, `charts`; My-Page owns the view and control rail. |
| `SONGS_VIEW_OPTIONS` / `songsView` | `songs`, `genre`; default songs. |
| `SONGS_SORT_OPTIONS` / `songsSort` | Profit descending/tie plays; plays descending/tie profit; streamed descending/tie plays; ratio descending/tie plays; recent descending; genre alphabetical/tie profit. Default profit. |
| `songsQuery` | Case-insensitive title, full genre label or network label; visible when focused. |
| `GENRE_METRIC_OPTIONS` / `genreMetric` | `profit`, `plays`, `streamed`; default profit. |
| `FEED_VIEW_OPTIONS` / `feedView` | `plays`, `settlements`; default plays. |
| `playsFilter` | `all`, `stream`, `license`; all includes both event kinds. |
| `settlementsFilter` | `all`, `settled`; candidate rows have hash or positive royalty/gross; settled further requires hash or positive royalty. The renderer labels these rows Settled. |

### Row data and filtering

The tables receive data through `Earn` props: `songs`, `songMetrics`, `events`, network/source/date filters and `onSelectSong`. [useCreatorEarnings](https://github.com/rawchunks/app/blob/main/src/hooks/use-earnings.ts#L364) requests `GET /api/earnings?creator={encoded address}` with `x-user-address: address`; it uses query key `['earnings', 'creator', address]`, enables requests for a nonempty address and passes the query's abort signal. It returns `CreatorEarningsResponse` or `null` on a failed HTTP response, with a 10-second stale time, 15-second polling and refresh on window focus. AG Grid receives the computed rows through `rowData`.

| Table row type | Fields |
| --- | --- |
| `NormalizedSongRow` | `song`, `songId`, `title`, `genre`, `allGenres`, `genreLabel`, `network`, `networkFamily`, `networkLabel`, `logoUrl`, `profitUsd`, `streamProfitUsd`, `licenseProfitUsd`, `platformFeeUsd`, `grossUsd`, `plays`, `rawPlays`, `licensesSold`, `licensesMinted`, `licensesRemaining`, `durationMs`, `streamedMs`, `ratioPct`, `txHash`, `explorerUrl`, `lastPlayedAt`. |
| `GenreSlice` | `genre`, `profitUsd`, `plays`, `streamedMs`, `ratioSum`, `songCount`, `value`, `avgRatioPct`, `color`, `pct`. |
| `EnrichedEventItem` | `id`, `songId`, `songTitle`, `genre`, `allGenres`, `genreLabel`, `network`, `networkFamily`, `networkLabel`, `kind`, `royaltiesUsd`, `platformFeeUsd`, `totalUsd`, `streamedMs`, `durationMs`, `ratioPct`, `txHash`, `explorerUrl`, `payer`, `timestamp`. |

Row amounts are USD numbers, durations use milliseconds, timestamps use Unix milliseconds and ratios use percentage points. `song` retains the original `Song` object. `txHash`, `explorerUrl`, `payer` and `logoUrl` allow `null`; `allGenres` is a string array.

`normalizedSongs` excludes missing metrics and the wrong network family, selects profit by revenue source, builds genre labels and derives gross/fee from the 80% creator share. `unifiedEvents` fills missing display fields, filters events by network/source/date and sorts newest first; it adds aggregate stream/license rows for songs without a remaining event. Those added rows follow the base-event date filter.

`sortedSongs` searches title, full genre label or network label. `genreSlices` gives each listed genre the song's full metric, averages ratio per song, sorts by selected metric/tie plays and assigns percentages/colors. `filteredPlayEvents` selects stream/license kinds; settlement candidates have a hash or positive royalty/gross, and the settled filter requires a hash or positive royalty.

### Grid styling

[Grid CSS](https://github.com/rawchunks/app/blob/main/src/styles/mypage.css#L3689) supplies `--ag-background-color`, `--ag-foreground-color`, header colors, border/row-border, hover/selected-row colors, font family/size and horizontal padding. It removes wrapper borders, hides header menu/filter icons, applies `cm-ag-right` alignment and removes activity headers.

Genre colors use `BRAND_PALETTE`: `#00ff41`, `#00ccff`, `#ff0055`, `#ffaa00`, `#00e5a3`, `#38bdf8`.

---

## What We Learned & What's Next

Building Rawchunks confirmed that frontier open-weights models (`Nemotron-3-Ultra-550b-a55b` and `Nemotron-3.5-Super-VL`) served over low-latency token infrastructure (Nebius Token Factory) can act as real-time creative state machines inside latency-sensitive media software when paired with deterministic local graphs and tool-level validators.

Next, we are expanding the Algorithmic Composer's directed graph to orchestrate multi-section song transitions automatically across timeline markers, adding collaborative multi-user studio sessions over WebSockets, and releasing SDK packages for third-party developers building on the Rawchunks API.

## Source and setup

| Repository | Contents |
| --- | --- |
| [app](https://github.com/rawchunks/app) | Studio, Editor, library and Market |
| [backend](https://github.com/rawchunks/backend) | Agents, APIs, audio runtime and scaling configuration |
| [blockchain](https://github.com/rawchunks/blockchain) | Song and license contracts on EVM and Solana |
| [docs](https://docs.rawchunks.space) | App guides, API reference and AI/ML documentation |
| [landing](https://github.com/rawchunks/landing) | Website |
| [demo](https://www.youtube.com/watch?v=7ICiGRS8b1M) | Project Presentation & Demo |


Run the app against the hosted API:

```sh
git clone https://github.com/rawchunks/app.git
cd app
npm ci
npm run dev
```

Open `http://localhost:3001` and connect your account. Set `VITE_API_URL` for your own backend; its [README](https://github.com/rawchunks/backend#readme) covers setup. For an integration, start with [API requests](https://docs.rawchunks.space/dev/guides/connect).
