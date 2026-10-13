# Exepad for Claude

Exepad turns a conversation with Claude into a working web app that you own. Claude writes the app's code in your
Exepad account, and Exepad keeps every change as a revision, builds it, gives you a private preview, and publishes a
build only after you approve it in Exepad.

## What it does

- Creates web apps in your Exepad account, and reads, searches and changes the source of the apps you connect.
- Builds each change and reports compiler errors, so Claude can fix them.
- Checks a build's private preview for console errors, exceptions and failed requests, and takes its screenshot.
- Where Claude shows Exepad's card in the conversation, the card holds the build's state, its latest screenshot and
  **Open preview**, which opens the private preview in Exepad.
- Publishes a build after you approve it in Exepad, and lists an app's releases.
- Renames an app, and unpublishes or archives one after you confirm it in Exepad.

The plugin holds one skill, `build-an-app`, which describes how Claude works with Exepad: reading Exepad's reference
first, changing the source in small steps, building and fixing until the build succeeds, checking the preview, and
asking you before anything goes live. It also names Exepad's server, `https://mcp.exepad.com/mcp`.

## How to use it

1. You need an Exepad account. Every Exepad plan connects through Claude's connector.
2. Adding the plugin does not add the connector or sign you in. In Claude, open the plugin from Customize › Plugins,
   select its Connectors tab, and add and connect Exepad there. You can also add it from Customize › Connectors:
   choose Add custom connector, name it Exepad and enter `https://mcp.exepad.com/mcp`.
3. Sign in with your Exepad account, and choose the apps Claude may work on.
4. Ask Claude, for example: "Build a habit tracker where I can tick off habits each day, then show me the preview."

On a Claude Team or Enterprise plan, an Owner adds Exepad in Organization settings › Connectors, and members then
connect with their own Exepad accounts.

## What it sends and stores

The plugin runs no code of its own and stores nothing. Through the Exepad connector, Claude sends app source, and
build, preview and publish requests, to your Exepad account at `mcp.exepad.com`, and reads back the results: file
contents, build reports, preview checks and screenshots. Claude works only on the apps you choose when you connect
and the apps it creates through that connection. Nothing goes live until you approve a publish in Exepad. Exepad's
privacy policy is at https://exepad.com/privacy.

## Claude Code

In Claude Code, the plugin's `exepad` server connects with its own sign-in, which needs Exepad Pro. On other Exepad
plans, switch the plugin's `exepad` server off in `/mcp` and use the Exepad connector you added on claude.ai, which
Claude Code offers when you sign in with the same Claude account.

## Help

- Setup and documentation: https://exepad.com/ai-assistant-plugins
- Support: https://exepad.com/contact
- Security reports: support@exepad.com

The plugin is released under the MIT License (see `LICENSE`).
