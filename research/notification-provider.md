# Which hosted provider should deliver Cadence's in-app Notifications?

Research for issue #10 (map: #9). Scope: **in-app only** (bell, feed, toasts). Email, push and SMS don't matter here. Sources were read on 2026-11-11. Pricing pages change often, so check them again before signing up.

> Note on the framework version: the ticket says React Router v7, but `package.json` pins `react-router` / `@react-router/*` at **8.3.0** with React **^19.2.7**. Every SDK below is a browser library that holds a WebSocket open, so the version doesn't change the outcome. What matters is React 19 support (checked below) and mounting the SDK only on the client.

## Recommendation

**Use Knock. Novu is the runner-up.**

- **Knock** has the most generous free tier that fits the rules: 10,000 delivered messages a month, unlimited environments, unlimited feed retention, and no hard cap. It also has the most complete in-app state model (seen, read, interacted, archived, set automatically by the feed), a first-party Node SDK with a built-in `signUserToken` helper, and a well-documented token model that suits DevUI user-switching: the server mints a short-lived RS256 JWT per user, and the client re-authenticates by calling `authenticate()` again. The cost: Knock ships no prebuilt toast component, so toasts are built from the feed's real-time event plus a toast library (Knock's own guide does this).
- **Novu** comes second. Its free tier also allows 10k workflow runs a month, and it ships a polished `<Inbox />` for React 19. Its feed auth is a simple HMAC-SHA256 of the `subscriberId`, which is even simpler to wire up for the DevUI. It loses to Knock on its Free-plan limits: **at most 5 workflows**, 3 team members, and only 24-hour activity-log retention. Cadence's v1 triggers cover four roles, so 5 workflows is a real constraint.

## Comparison

| Criterion | Knock | Novu (Cloud) | MagicBell | Courier | SuprSend |
| --- | --- | --- | --- | --- | --- |
| Free tier | 10,000 messages/mo; unlimited environments, workflows, feeds, members; unlimited feed retention; 30-day logs; no hard cap [1] | 10,000 workflow runs/mo; **5 workflows**; 2 environments (Dev+Prod); 3 members; 24h activity retention; unlimited subscribers [2] | **1,000 deliveries/mo**; 1 project [3] | 10,000 sends/mo; overage behaviour not stated; Inbox on Free not explicitly stated [4] | 10,000 notifications/mo; SuprSend branding; 30-day logs; no batching/digest or preferences on Free; no hard cut-off [5] |
| React SDK (React 19) | `@knocklabs/react` 0.15.1, peer `react ^17 \|\| ^18 \|\| ^19`; feed popover, bell button, full-page feed, headless hooks; **no prebuilt toast** [6][7][8] | `@novu/react` 3.19.2, peer `^18 \|\| ^19`; `<Inbox />` (Bell, Notifications, Preferences) + headless hooks; `routerPush` prop for React Router navigation [8][9][10] | `@magicbell/react` 1.14.0; Inbox, FloatingInbox, ToastContainer [8][11] | `@trycourier/courier-react` 9.3.5, peer `>=18`; CourierInbox, CourierInboxPopupMenu, CourierToast, `useCourier()` [8][12] | `@suprsend/react` 1.5.0; Inbox, NotificationFeed, toasts [8][13][14] |
| SSR | No SSR guidance in docs; treat as client-only | No SSR guidance; treat as client-only | Not documented | Documented as "client-side only" [12] | Not documented |
| Official Node SDK | `@knocklabs/node` 1.41.1: `workflows.trigger(key, { recipients, data })` [8][15] | `@novu/api` 3.19.1: `novu.trigger({ to: { subscriberId }, workflowId, payload })` [8][16] | `magicbell-js` 1.8.0 ("Browser/Node Client"); a dedicated server SDK is less clearly documented [8][11] | `@trycourier/courier` 10.0.0 [8] | `@suprsend/node-sdk` 1.15.1 [8] |
| Hosted | Yes | Yes (self-host optional, not needed) [2] | Yes | Yes | Yes |
| Real-time feed | Yes, socket; auto-disconnects inactive tabs; `items.received.realtime` event [7][17] | Yes, real-time unread count and sync [9] | Yes, "persistent connection" [18] | Yes, WebSocket via `listenForUpdates()` [12] | Yes (feed + toasts) [14] |
| Read/unread/seen | seen, read, interacted, archived (automatic in feed) [19] | unread/read, archive, snooze [9] | read/unread, archive; no separate "seen" [18] | read, opened, archived, unread badges [20] | read/seen/archived (drop-in feed) [14] |
| Recipient id | Your user id string (`recipients: ['user-id']`); users can be identified inline [15] | `subscriberId` (your id); auto-created on first use [10][16] | `user_external_id` and/or `user_email` [21] | Your `user_id` [22] | `distinct_id` [23] |
| Feed auth | Public key + **RS256 JWT** (`sub` = user id), signed server-side with a signing key; required in "enhanced security mode"; `signUserToken` in Node SDK, default 1h expiry; `onUserTokenExpiring` callback [24] | Public `applicationIdentifier` + **`subscriberHash` = HMAC-SHA256(secretKey, subscriberId)**, hex; Inbox refuses to load without it once enabled [25] | **HS256 user JWT** signed with project secret; claims `user_external_id`/`user_email`, `api_key` [21] | **JWT from `POST /auth/issue-token`** with scope `user_id:<id> inbox:read:messages inbox:write:events`; SDK doesn't refresh [22] | Public key + **ES256 JWT** (`entity_type: subscriber`, `entity_id`, `exp`, `scope.tenant_id`); `refreshUserToken` callback [23][13] |

## Per-provider notes

### Knock (recommended)

- **Free tier fit.** Only messages delivered on a channel count toward the 10k [1]. With in-app only, that means one count per Notification per recipient. Unlimited environments make separate dev, test and prod straightforward. Going over the limit doesn't stop sending; Knock asks you to add billing the next month [1].
- **Security model.** Without enhanced security, "a request with your public key can act as any user" [24]. In production, turn on enhanced security mode and have the Cadence server sign a token per session user. In the Node SDK that's `signUserToken(userId, { expiresInSeconds })`, which reads `KNOCK_SIGNING_KEY` [24][15]. The client passes it as `userToken` to the provider. "Calling `authenticate` again with a new token restarts the real-time connection" [24].
- **DevUI implications.** When the DevUI switches users, the server session changes. A loader can return `{ knockUserId, knockToken }` for the current session user, and the client re-keys the provider. Knock users are keyed by Cadence's own user id strings, so nothing has to be pre-provisioned. Users can be created inline when a workflow is triggered [15].
- **UI.** `NotificationIconButton` with `NotificationFeedPopover` covers the bell and feed. Badge counts can use `unseen` (the default) or `unread` [7]. Toasts are built on `feedClient.on("items.received.realtime", ...)` plus a toast library, because "there are no out-of-the-box toast components in the `@knocklabs/react` library" [17].
- **SSR.** Knock's docs don't cover SSR. The components use hooks and sockets, so render them client-only. Don't instantiate the client in route modules on the server.

### Novu (runner-up)

- **Strengths.** It's the most "batteries included" inbox. HMAC auth is the easiest to wire up: one `createHmac('sha256', NOVU_SECRET_KEY).update(subscriberId).digest('hex')` per request [25]. `routerPush` plugs into React Router navigation [10].
- **Free-tier limits that matter.** At most 5 workflows, 3 team members, and only 24h activity-feed (debug log) retention [2]. The pricing page lists In-app as a Free channel but doesn't spell out whether `<Inbox />` is on Free [2].
- **Risk.** The HMAC never expires, so a leaked hash grants that subscriber's feed until the secret is rotated. Knock's and Courier's JWTs have an `exp`.

### Courier

- Good React components (including a real `CourierToast`). It's the only provider that explicitly documents Next.js support as client-side only [12].
- The free 10k sends/mo is fine, but the pricing page doesn't confirm Inbox is on the Developer plan or what happens above the limit (the FAQ answer is blank) [4].
- The SDK doesn't refresh tokens, so the app has to re-mint and call `signIn` itself [22].
- Its product focus is cross-channel orchestration, which Cadence doesn't need.

### SuprSend

- 10k/mo free with a real in-app feed, toasts, and ES256 JWT auth with a refresh callback [5][23].
- Free lacks batching/digest and preferences (preferences are out of scope for v1 anyway) [5]. The docs site blocked automated fetching (HTTP 403) during this research, so less of it could be verified first-hand.
- Its multi-tenant JWT `scope` adds a concept Cadence doesn't use. Omit it to use the default tenant [23].

### MagicBell

- Ruled out on the free tier: **1,000 deliveries/mo** and one project [3], which doesn't cover dev plus a small production load. The next plan costs $249/mo [3].
- Its HS256 user JWT examples use a one-year expiry [21], so short-lived tokens would be up to Cadence.

## What this means for the open questions in #9

- **Identity and the DevUI.** With Knock, Cadence's user id is the recipient id. The server signs a short-lived user token per request or session, and the client re-authenticates on user switch. Enhanced security can stay off in the development environment and on in production, so the token path is exercised wherever it matters.
- **Failure must not block the action.** `workflows.trigger` is one HTTP call from the server. Wrap it so errors are logged and swallowed, or queue it. Knock doesn't stop processing over the free limit [1].
- **Local dev and tests.** Knock's unlimited environments let tests use a dedicated environment, or a fake behind a small `notify()` interface. That decision belongs to the later ticket.

## Sources

1. Knock pricing — https://knock.app/pricing
2. Novu pricing — https://novu.co/pricing
3. MagicBell pricing — https://www.magicbell.com/pricing
4. Courier pricing — https://www.courier.com/pricing
5. SuprSend pricing — https://www.suprsend.com/pricing
6. Knock React overview — https://docs.knock.app/in-app-ui/react/overview
7. Knock React feed guide — https://docs.knock.app/in-app-ui/react/feed
8. npm registry metadata (versions, React peer ranges), fetched 2026-11-11 — `https://registry.npmjs.org/<package>/latest` for each package named above
9. Novu Inbox overview — https://docs.novu.co/platform/inbox/overview
10. Novu React quickstart — https://docs.novu.co/platform/quickstart/react
11. MagicBell React components — https://www.magicbell.com/docs/libraries/magicbell-react
12. Courier React SDK — https://www.courier.com/docs/sdk-libraries/courier-react-web
13. SuprSend React SDK README — https://github.com/suprsend/suprsend-react-sdk
14. SuprSend React inbox overview — https://docs.suprsend.com/docs/inbox-react
15. Knock Node SDK README — https://github.com/knocklabs/knock-node
16. Novu TypeScript server SDK — https://docs.novu.co/platform/sdks/server/typescript
17. Knock real-time toasts guide — https://docs.knock.app/in-app-ui/react/toasts
18. MagicBell in-app channel — https://www.magicbell.com/docs/channels/in_app
19. Knock message statuses — https://docs.knock.app/send-notifications/message-statuses
20. Courier Inbox overview — https://www.courier.com/docs/platform/inbox/inbox-overview
21. MagicBell User JWT — https://www.magicbell.com/docs/api/authentication/user
22. Courier Inbox authentication — https://www.courier.com/docs/platform/inbox/authentication
23. SuprSend client authentication — https://docs.suprsend.com/docs/client-authentication
24. Knock security and authentication — https://docs.knock.app/in-app-ui/security-and-authentication
25. Novu Inbox: prepare for production (HMAC) — https://docs.novu.co/platform/inbox/prepare-for-production
