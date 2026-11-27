<img src="https://raw.githubusercontent.com/uttertalk/brand/main/cover.png" alt="Utter: your users ask, your app changes">

# Utter docs

With [Utter](https://www.utter.talk), your users ask and your app changes for them. They can change how it looks, just
for themselves, and they can ask for work, which Utter does inside your app, as them, with only the actions you allow.
Out loud or typed. Anything that changes data waits for a yes.

These are the docs as plain files. The same pages, with copy buttons, are at
[app.utter.talk/setup](https://app.utter.talk/setup) and [app.utter.talk/docs](https://app.utter.talk/docs).

| page | for |
|---|---|
| [SETUP.md](SETUP.md) | Adding Utter to your app step by step, with the exact prompts to give your AI coding agent, common problems, and the same setup by hand. |
| [REFERENCE.md](REFERENCE.md) | `createUtter`, the action list, `<Utter />`, hooks, events, the command line, safety, limits and the protocol. |

## Quick start

```bash
npx utter-sdk init
```

Then ask your AI coding agent (Claude Code, Cursor or Codex):

```
Set up Utter in this app. Follow the utter skill.
```

Check it with `npx utter-sdk check`, add your own Claude or Gemini key in the
[dashboard](https://app.utter.talk/dashboard), and switch Utter on.

## More

- Package: [uttertalk/sdk](https://github.com/uttertalk/sdk) · [npm](https://www.npmjs.com/package/utter-sdk)
- Example app: [uttertalk/example-app](https://github.com/uttertalk/example-app)
- Every prompt and safety rule: [uttertalk/brain](https://github.com/uttertalk/brain)
- For AI coding agents: [utter.talk/llms.txt](https://www.utter.talk/llms.txt), and all docs in one file at
  [utter.talk/llms-full.txt](https://www.utter.talk/llms-full.txt)
- Found a mistake in the docs? Open an issue or a pull request here.

## License

[MIT](LICENSE)
