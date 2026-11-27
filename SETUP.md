# How to set it up

About 15 minutes with an AI coding agent. You run one command, paste one prompt, check what it wrote, and switch Utter
on.

1. [Before you start](#1-before-you-start)
2. [Run the setup command](#2-run-the-setup-command)
3. [Ask your AI to finish it](#3-ask-your-ai-to-finish-it)
4. [Check its work](#4-check-its-work)
5. [Run the check](#5-run-the-check)
6. [Try it yourself](#6-try-it-yourself)
7. [Switch it on](#7-switch-it-on)
- [Prompt library](#prompt-library)
- [Common problems](#common-problems)
- [By hand](#by-hand)

## 1. Before you start

You need:

- An app with a JavaScript server and React screens. The setup command writes files for Next.js (app router); other
  frameworks work too, you just serve one route yourself (see [By hand](#by-hand)).
- Your own login. Utter always acts as the person who is signed in to your app, inside their own company, so it needs
  to know who that is.
- An AI coding agent: Claude Code, Cursor or Codex. Without one, plan on about 30 minutes doing the same steps by hand.
- A Claude or Gemini API key for Utter to answer with. You add it in the dashboard, and you pay your provider directly.
  No key? [Get a Gemini key from Google AI Studio](https://aistudio.google.com/apikey).
- An Utter account. [Make one here](https://app.utter.talk/signup); the setup command can also make the project for
  you.

## 2. Run the setup command

In your app's folder:

```bash
npx utter-sdk init
```

What happens:

- Your browser opens on an Utter page. Sign in, check that it shows the same short code as your terminal, and pick a
  project (or make a new one with your app's name and address).
- The terminal saves your project's keys in `.env.local` (`UTTER_PROJECT` and `UTTER_SECRET`) and makes sure that file
  stays out of git.
- It installs `utter-sdk`, creates `lib/utter.ts` and the route `app/api/utter/[...path]/route.ts`, and adds
  instructions for your coding agent in `.claude/skills/utter`, `.agents/skills/utter` and `.cursor/rules/utter.mdc`.

It ends with something like this:

```
Utter setup

  ✓ Connected to Acme CRM: keys saved in .env.local
  ✓ Added .env.local to .gitignore, so the secret stays out of git
  ✓ Installed utter-sdk
  ✓ Created lib/utter.ts
  ✓ Created app/api/utter/[...path]/route.ts
  ✓ Added the Utter skill for Claude Code, Codex and Cursor

Next: ask your AI coding agent to finish it
```

## 3. Ask your AI to finish it

Open your coding agent in the same folder and paste:

```
Set up Utter in this app. Follow the utter skill.
```

It works through the skill step by step:

- Reads your code: your data, how companies and people are stored, your login, your screens.
- Writes the **action list** in `lib/utter.ts`: the things Utter may read and change, each one a small function that
  uses your own data layer and only ever touches one company's data.
- Connects your login, so Utter knows who is speaking and which company they belong to.
- Puts the Utter button in your signed-in layout, with two or three example requests that fit your app.
- Makes your pages refresh after Utter changes something.
- Runs the check and tells you what it did and what it left out.

If your agent doesn't pick up the skill on its own, use the long version:

```
Read .claude/skills/utter/SKILL.md (or .agents/skills/utter/SKILL.md) and follow it step by step to set up Utter in this app: fill in lib/utter.ts with the action list, the actions, isMember and getViewer, put the <Utter /> button in the signed-in layout, refresh the page's data on the "utter:changed" event, then run npx utter-sdk check and fix what it reports. Ask me if anything about our data or login is unclear. When you're done, list the actions you added and anything you left out on purpose.
```

## 4. Check its work

Look over the change like any other pull request. Most of it is in `lib/utter.ts`. A good action list is short, reads
first, and changes only a few safe things:

```ts
actions: [
  { name: "leads.list", description: "The company's leads, newest first, optionally filtered by status or a search", args: "(opts?: { status?: string; q?: string })", returns: "Lead[]" },
  { name: "leads.get", description: "One lead", args: "(id: string)", returns: "Lead | null" },
  { name: "users.list", description: "The company's team", args: "()", returns: "User[]" },
  { name: "leads.update", description: "Change a lead's status, value or owner", args: "(id: string, patch: {...})", returns: "Lead", writes: true },
  { name: "notes.add", description: "Add a note to a lead", args: "(leadId: string, text: string)", returns: "Note", writes: true },
]
```

Red flags to fix before you go on:

- **An action that doesn't filter by tenantId.** Every read and every write has to stay inside `{ tenantId }`, the
  company of the person speaking. A company id must never come from the arguments.
- **An action that changes data without `writes: true`.** Without it, Utter treats the action as reading and runs it
  without asking.
- **Deleting, billing, users and permissions, or emailing people outside the company.** Leave these out unless you
  really want them done by voice.
- **Secrets in the types.** Password hashes, tokens, card numbers or internal notes don't belong in what actions return.
- **isMember that always says yes.** It has to really look up whether that person belongs to that company.

## 5. Run the check

Start your app, then:

```bash
npx utter-sdk check --app http://localhost:3000 --tenant <a company id> --user <one of its people>
```

It checks your keys, reads your action list the way Utter does (signed with your secret), and runs each reading action
for that company and person. All good looks like:

```
Utter check

  ✓ Keys work for Acme CRM
  · Utter calls your app at https://app.acme.com (change it in the dashboard's Settings)
  · Not switched on yet: your customers don't see the Utter button
  ✓ The action list is complete: 5 actions, 2 of them change data
  ✓ leads.list works (128 rows)
  ✓ users.list works (6 rows)

All good.
```

If it reports problems, paste the output into your agent with the prompt [The check failed](#the-check-failed).

## 6. Try it yourself

Utter calls your app from its own servers, so it needs your app's public address. Set it in the dashboard under
Settings. For local testing, a tunnel (like `ngrok http 3000` or `cloudflared tunnel`) gives your laptop a public
address.

Switch Utter on (next step), open your app signed in, press the button in the corner and try:

- A question about your data: "How many open deals do we have?"
- Something it has to look up first: "Who owns the biggest lead?"
- A change: "Assign every unassigned lead to me." Utter lists the changes and waits. Say "yes" or press the button.

Every request, what Utter read and every change it made shows up in the dashboard under Activity.

## 7. Switch it on

In the [dashboard](https://app.utter.talk/dashboard), your project shows a checklist: Utter has an AI (add your own
Claude or Gemini key under AI), your app answers Utter, your actions are listed, your screens reach Utter. When it's all done, press Switch on. The button appears for everyone in
your app at once. Switch it off any time to hide it again.

Want to try it before anyone else sees it? While it's switched off, open your app signed in as yourself, then press
Let test next to your name under Test it first. Only the people you pick see the button until you switch it on.

You can also switch single actions off there. Utter's AI is never told about an action that's off, and Utter refuses to
run it.

## Prompt library

Paste these into your coding agent for common changes. Each one works on its own.

### Set up from scratch

```
Set up Utter in this app. Follow the utter skill.
```

### Add more actions

```
Add these to Utter's action list in lib/utter.ts: [describe what people should be able to ask or do, e.g. "see invoices and their status", "mark a task done"]. Follow the utter skill's rules: filter by tenantId, check the arguments, and mark anything that changes data with writes: true. Then run npx utter-sdk check.
```

### Let Utter answer questions about something

```
Make Utter able to answer questions about [our projects / invoices / tickets]. Add reading actions to lib/utter.ts (a list with useful filters, and a get by id), with the fields people would ask about in the types, and nothing secret. Follow the utter skill.
```

### Remove anything that deletes

```
Go through Utter's action list in lib/utter.ts and remove every action that deletes or archives data, or that changes users, roles, permissions, billing or payments. List what you removed.
```

### Check it's safe

```
Review lib/utter.ts for safety: every action must only read or change data of the company in tenantId, never take a company id from its arguments, check its arguments' types and allowed fields, and every action that changes data must have writes: true. isMember must really check membership. Fix anything that's wrong and tell me what you changed.
```

### Utter says "Not signed in"

```
The Utter button in this app says "Not signed in" (or never shows up) even though I'm signed in. Look at getViewer in lib/utter.ts: it must return the signed-in person's company and user from our own login, using the same helper our API routes use. Check it works inside app/api/utter/[...path]/route.ts and fix it.
```

### The check failed

```
npx utter-sdk check reported these problems:

[paste the output here]

Fix them in lib/utter.ts (or the env file / route if that's what it says), then run the check again until it says All good.
```

### Refresh my screens after a change

```
After Utter changes something, the page doesn't show it until I reload. Listen for the "utter:changed" window event where the Utter button is, and refresh the data on screen: router.refresh() for server components, or invalidate the queries we use (SWR mutate / React Query invalidateQueries) for client-side data.
```

### Match our brand colours

```
Make the Utter button match our design: set the CSS variables --utter-accent, --utter-accent-ink, --utter-bg, --utter-ink, --utter-muted, --utter-line and --utter-soft on a parent of <Utter /> using our own colour tokens.
```

### Move the button to the left

```
Move the Utter button to the bottom left: pass position="bottom-left" to <Utter />.
```

### Better example requests

```
Change the examples on <Utter /> to two or three short requests our users would really say, that work with the actions in lib/utter.ts.
```

## Common problems

### The button doesn't show up

- Utter isn't switched on yet: check the dashboard. The button stays hidden until it is.
- You're not signed in to your app, or `getViewer` returns null. Use the prompt
  [Utter says "Not signed in"](#utter-says-not-signed-in).
- Open `/api/utter/status` in your app while signed in. It should show `{"live":true,...}`; if not, it says what's
  wrong.

### "Your app turned Utter away" or a 401 in the check

`UTTER_SECRET` in your app doesn't match the dashboard. Copy it again from the dashboard, and restart your app after
changing env files. If you made a new secret, the old one stopped working right away.

### "Couldn't reach your app"

Utter calls the address in the dashboard's Settings, from the internet. A `localhost` address only works with a
tunnel; in production it has to be your public https address.

### The mic button is missing

Listening uses the browser's own speech recognition, which Chrome, Edge and Safari have and Firefox doesn't. Without it
the mic hides and typing still works. The browser also asks for microphone permission the first time.

### Utter says it can't do something

It can only use the actions on your list. Add the action (prompt [Add more actions](#add-more-actions)), or check it
isn't switched off in the dashboard.

### It changed something, but the page still shows the old data

Use the prompt [Refresh my screens after a change](#refresh-my-screens-after-a-change).

## By hand

The same setup without an AI agent.

### 1. Install and add your keys

```bash
npm install utter-sdk
```

In `.env.local` (keep it out of git), from your project in the dashboard:

```
UTTER_PROJECT=prj_...
UTTER_SECRET=sk_...
```

### 2. lib/utter.ts

```ts
import { createUtter } from "utter-sdk/server";
import { db } from "./db";
import { auth } from "./auth";

export const utter = createUtter({
  manifest: {
    app: "Acme CRM",
    description: "A CRM for small sales teams.",
    types: `type Lead = { id: string; name: string; company: string; value: number; status: "new" | "qualified" | "won" | "lost"; ownerId: string | null }
type User = { id: string; name: string }`,
    actions: [
      { name: "leads.list", description: "The company's leads, newest first, optionally filtered by status", args: "(opts?: { status?: string })", returns: "Lead[]" },
      { name: "users.list", description: "The company's team", args: "()", returns: "User[]" },
      { name: "leads.update", description: "Change a lead's status or owner", args: "(id: string, patch: { status?: string; ownerId?: string | null })", returns: "Lead", writes: true },
    ],
  },

  // Each action gets the company (tenantId) and person (userId) first, then its arguments.
  actions: {
    "leads.list": ({ tenantId }, opts) =>
      db.lead.findMany({ where: { orgId: tenantId, status: opts?.status }, orderBy: { createdAt: "desc" }, take: 300 }),
    "users.list": ({ tenantId }) => db.user.findMany({ where: { orgId: tenantId }, select: { id: true, name: true } }),
    "leads.update": ({ tenantId }, id, patch) =>
      db.lead.update({
        where: { id: String(id), orgId: tenantId },
        data: { status: patch?.status, ownerId: patch?.ownerId },
      }),
  },

  isMember: async (tenantId, userId) => Boolean(await db.user.findFirst({ where: { id: userId, orgId: tenantId } })),

  getViewer: async () => {
    const session = await auth();
    if (!session?.user) return null;
    return {
      tenant: { id: session.user.orgId, name: session.user.orgName },
      user: { id: session.user.id, name: session.user.name },
    };
  },
});
```

### 3. The route

```ts
// app/api/utter/[...path]/route.ts
import { utter } from "@/lib/utter";

export const { GET, POST } = utter.handler;
```

Not on Next.js? Serve `utter.handler.GET` and `utter.handler.POST` at `/api/utter/*`. They take a standard `Request`
and a second argument `{ params: Promise.resolve({ path: [...] }) }` with the rest of the path, and return a standard
`Response`.

### 4. The button

```tsx
"use client";
import { useEffect } from "react";
import { useRouter } from "next/navigation";
import { Utter, CHANGED_EVENT } from "utter-sdk/react";

export function UtterButton() {
  const router = useRouter();
  useEffect(() => {
    const refresh = () => router.refresh();
    window.addEventListener(CHANGED_EVENT, refresh);
    return () => window.removeEventListener(CHANGED_EVENT, refresh);
  }, [router]);
  return <Utter app="Acme CRM" examples={["Which deals close this month?", "Assign my unassigned leads to me"]} />;
}
```

Put `<UtterButton />` once in your signed-in layout. Then run the check and switch Utter on.
