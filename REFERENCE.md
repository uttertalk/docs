# Reference

Reference for `utter-sdk`. New here? Start with [SETUP.md](SETUP.md).

- [How it works](#how-it-works)
- [createUtter](#createutter)
- [The action list](#the-action-list)
- [Actions](#actions)
- [getViewer and isMember](#getviewer-and-ismember)
- [`<Utter />`](#utter-)
- [Their own look](#their-own-look)
- [Hooks](#hooks)
- [Events](#events)
- [Command line](#command-line)
- [Safety](#safety)
- [Limits](#limits)
- [Your team](#your-team)
- [Protocol](#protocol)

## How it works

- A signed-in person presses the Utter button in your app and speaks (or types). The browser turns speech into text.
- The button sends the text to your own route, `/api/utter/ask`. Your route knows who is signed in through your login,
  and passes the request on to Utter with a short session for that person.
- Utter's AI gets your action list and the request. It runs reading actions right away by calling your app back (signed
  with your secret), as that person, inside their company.
- Actions that change data are never run at this point. Utter returns a plan listing every change, signed by Utter, and
  the person says yes or no.
- On yes, Utter checks the plan and runs exactly those steps, once. The reply is spoken and shown, and logged in your
  dashboard.

## createUtter

On your server, once:

```ts
import { createUtter } from "utter-sdk/server";

export const utter = createUtter({
  manifest,      // the action list (below)
  actions,       // one function per action
  isMember,      // (tenantId, userId) => boolean
  getViewer,     // (req) => { tenant, user } | null
  // url, project, secret: optional, default to UTTER_URL, UTTER_PROJECT, UTTER_SECRET
});

// app/api/utter/[...path]/route.ts
export const { GET, POST } = utter.handler;
```

`utter.handler` answers Utter's signed calls (`GET /manifest`, `POST /call`) and the button's calls (`GET /status`,
`POST /ask`, `POST /confirm`, `POST /cancel`, `GET /activity`).

`utter.client(viewer)` makes the same calls from your own server code, for example to build your own interface:
`status()`, `ask(conversation)`, `confirm(plan, sig)`, `cancel(plan, sig)`, `activity()`.

## The action list

```ts
manifest: {
  app: "Acme CRM",
  description: "A CRM for small sales teams.",
  types: "type Lead = { id: string; name: string; status: string; ownerId: string | null }",
  actions: [
    { name: "leads.list", description: "The company's leads, newest first", args: "(opts?: { status?: string })", returns: "Lead[]" },
    { name: "leads.update", description: "Change a lead's status or owner", args: "(id: string, patch: { status?: string; ownerId?: string | null })", returns: "Lead", writes: true },
  ],
}
```

- `app`, `description`: your product's name and what it's for, in a sentence or two.
- `types`: TypeScript types of what your actions take and return, as plain JSON (dates as ISO strings). Only useful,
  safe fields.
- `actions[].name`: `thing.verb`, unique. `description`: one plain sentence. `args`: the argument list in parentheses,
  `"()"` for none. `returns`: the type it returns.
- `writes: true` on every action that changes data. Those only ever run after a confirmed plan.

## Actions

One function per action name. The first argument is always the context, then the arguments Utter passes:

```ts
actions: {
  "leads.list": ({ tenantId, userId }, opts) => ...,
  "leads.update": ({ tenantId, userId }, id, patch) => ...,
}
```

- `tenantId` is the person's company, `userId` the person. Always filter by `tenantId`; never take a company id from
  the arguments.
- Arguments come from the AI: check their types and allowed fields like any user input.
- Return plain JSON. Throw an `Error` with a plain message when something's wrong; Utter reads it out.
- Only actions that are both on the list and in `actions` can run. Anything else is refused.

## getViewer and isMember

```ts
getViewer: async (req: Request) => ({
  tenant: { id: "org_123", name: "Greenfield Solar" },
  user: { id: "u_42", name: "Sarah Chen", title: "Sales Manager" },   // title is optional
}),   // or null when nobody is signed in

isMember: async (tenantId: string, userId: string) => boolean
```

`getViewer` runs on every call from the button and uses your own login. `isMember` runs on every call from Utter: your
app refuses to act for anyone who isn't really part of that company.

## `<Utter />`

```tsx
import { Utter } from "utter-sdk/react";

<Utter
  app="Acme CRM"                       // shown in the panel; defaults to manifest.app
  examples={["Which deals close this month?"]}
  position="bottom-right"              // or "bottom-left"
  voice                                // false: never speak replies
  lang="en-US"                         // defaults to the page's language
  basePath="/api/utter"                // where your route is
/>
```

- Hidden until Utter is switched on in the dashboard (testers you pick see it before that), and when nobody is signed in.
- Press the mic and speak; it sends when you stop. Or type. Replies are spoken (people can mute it; the choice is
  remembered in their browser).
- A plan shows every change with Yes and No buttons. Saying or typing "yes" / "no" works too. Anything else drops the
  plan and starts a new request.
- Escape closes the panel.

### Colours

Set these on a parent element:

```
--utter-accent, --utter-accent-ink, --utter-bg, --utter-ink, --utter-muted, --utter-line, --utter-soft
```

Every element has a class starting with `utter-` for finer changes.

## Their own look

People can change how your app looks, for themselves: "make it dark", "bigger text", "hide the Source column".
Nothing to set up: `<Utter />` does it from version 0.2.

- Utter's AI writes a style sheet for that one person. It's saved with Utter and applied on every page, on every
  device they use.
- Styling only. It never changes data or what your app can do, and can't load anything from elsewhere.
- To write it, the button sends an outline of the page with each request: tags, classes, and the words on headings,
  labels, buttons, menu links and column headers. Never what's in table cells or form fields.
- "Undo that" and "reset my look" always work. A look that would hide the Utter button is undone right away.
- Don't want it? Switch it off on your project's page in the dashboard. Everyone gets your app's own look back right
  away.

## Hooks

For your own look, from `utter-sdk/react`:

```ts
const status = useUtterStatus();   // { live, app, looks } or null
useLook({ enabled: status ? Boolean(status.live && status.looks) : undefined });   // keeps their own look on the page

const { turns, state, pending, send, confirm, cancel, reset } = useUtter({ page: Boolean(status?.looks) });
// turns: [{ id, role, text, list?, confirm?: { steps, state }, error? }]
// state: "idle" | "thinking";  pending: the plan waiting for a yes, or null
// send(text), confirm(), cancel();  page: send the page outline so they can change their own look

const { canListen, canSpeak, state, interim, error, listen, stopListening, speak, stopSpeaking } = useVoice();
// listen(): Promise<string> with what was said; speak(text): Promise<void>

yesOrNo("yeah do it")   // "yes" | "no" | null
```

## Events

After a confirmed plan changed something, the page gets `utter:changed` on `window` (also exported as
`CHANGED_EVENT`). Refresh your data on it:

```ts
window.addEventListener("utter:changed", () => router.refresh());
```

After someone changed their own look, it gets `utter:look` (`LOOK_EVENT`).

## Command line

```
npx utter-sdk init     # sign in, save keys, install, add lib/utter.ts, the route and the agent skill
  --relogin              pick a project again even if keys are saved
  --no-open              print the sign-in link instead of opening the browser
  --no-install           don't install the package
  --no-skill             don't add the agent skill files

npx utter-sdk check    # check keys and the running app's action list
  --app <url>            where your app runs (default http://localhost:3000)
  --tenant <id> --user <id>   also run each reading action for this company and person

--url <url>              a different Utter address
```

## Safety

Enforced by Utter's service and your route, not by asking the AI nicely:

- Only actions on your list, and never ones you switched off in the dashboard.
- Always as the signed-in person, inside their company: your `isMember` is checked on every call.
- Nothing that changes data runs without a yes. The plan is signed by Utter, made for one person, expires after 5
  minutes and runs once. A changed, expired, reused or someone else's plan is refused.
- Every call between Utter and your app is signed with your project secret (HMAC-SHA256) and refused after 5 minutes.
- Limits per request: 8 reads, 100 changes per plan.
- Every request is logged for 30 days: what was asked, what was read, what changed, and whether it worked.

## Limits

- Utter needs an AI to answer with. Add your own Claude or Gemini key in the dashboard: it's free, and you pay your
  provider directly, with no Utter limit.
- AI included ($99 a month, coming soon): Utter supplies the AI, with 10,000 requests a month per project.
- Per customer company: a daily number you set in the dashboard (0 turns Utter off for everyone).
- A request is one thing someone says or types. Saying yes to a plan doesn't count.

## Your team

Invite people to help run a project from its page in the dashboard, under Team.

- Only the project's owner can invite and remove people. Up to 20 people per project, invites included.
- The invite is an email with a link. It works once, for 7 days, and only for an account with the invited email, so a
  forwarded link is no use to anyone else.
- People on the team can do everything in the project: see its keys, switch Utter on or off, pick testers, change
  settings and the AI key, read the activity log and connect an app with `npx utter-sdk init`. Only the owner can
  delete the project and manage billing.
- Anyone on the team can leave. Removed people lose access right away.

## Protocol

What `utter-sdk` does for you, if you need to build it yourself:

```
# Your server gets a one-hour session for the signed-in person
POST {UTTER_URL}/api/v1/sessions
  Authorization: Bearer <UTTER_SECRET>
  X-Utter-Project: <UTTER_PROJECT>
  { "tenant": { "id", "name" }, "user": { "id", "name", "title"? } }
  -> { "token", "expiresAt" }

# With that token (Authorization: Bearer <token>)
GET  /api/v1/status                      -> { live, app, looks }
POST /api/v1/ask      { conversation, page? }  -> { kind: "answer" | "question" | "confirm" | "look" | "error", text, ... }
GET  /api/v1/look                        -> { css }   this person's own look
POST /api/v1/undo-look                   -> { css }   the look before their last change
POST /api/v1/confirm  { plan, sig }      -> { kind: "answer", text, done, failed }
POST /api/v1/cancel   { plan, sig }
GET  /api/v1/activity?before=&limit=     -> { entries, next }

# Utter calls your app, signed:
#   X-Utter-Timestamp: <ms>
#   X-Utter-Signature: hex HMAC-SHA256 of "<timestamp>.<body>" with UTTER_SECRET ("" body for GET)
GET  {your app}/api/utter/manifest       -> the action list
POST {your app}/api/utter/call  { tenantId, userId, op, args }  -> { ok, result } | { ok: false, error }
```

`UTTER_URL` defaults to `https://app.utter.talk`.
