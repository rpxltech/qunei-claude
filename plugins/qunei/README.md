# Qunei for Claude

Qunei keeps your books, and Claude does the bookkeeping. This plugin
brings Qunei into Claude in one install: Qunei’s skills for setting up a
ledger, coding bank statements, month-end reviews, GST and reports, and
the connection to your Qunei workspace. Nothing is posted to your books
until you approve it.

## Get started in Claude

You need a Qunei workspace (start at https://qunei.ai) and a paid Claude
plan.

1. In Claude, open **Customize**, then **Plugins**. Click **Add**, then **Add marketplace**, and enter:
   `rpxltech/qunei-claude`
2. Select **Qunei** and click **Add**.
3. Open Qunei’s **Connectors** tab. If Qunei shows **Not added**, add it, or on a Team or Enterprise plan ask an owner to add it. Then click **Connect**, sign in to Qunei and approve access to your workspace.
4. Open a new chat and say “Hi Qunei”.

## On a Claude Team or Enterprise plan

On these plans an owner adds Qunei for everyone under **Organization settings**, then **Plugins & skills**, and adds its connector under **Organization settings**, then **Connectors**. Each member then opens Qunei’s **Connectors** tab, clicks **Connect** and signs in to Qunei with their own account.

## Get started in Claude Code

Added Qunei in the Claude app on a Pro or Max plan? Claude Code, signed in to the same account, downloads it when it next starts. Then type **/reload-plugins**, or start Claude Code again, to load it. Otherwise, run this in a terminal:

```bash
claude plugin marketplace add rpxltech/qunei-claude && claude plugin install qunei@qunei && claude "/qunei:start"
```

In Windows PowerShell 5.1, run the three commands one at a time:

```powershell
claude plugin marketplace add rpxltech/qunei-claude
claude plugin install qunei@qunei
claude "/qunei:start"
```

To sign in, type **/mcp**, choose **plugin:qunei:qunei** and sign in to Qunei in your browser.

## On Claude’s free plan

Plugins need a paid Claude plan. On the free plan, Qunei works as a connector, and you can add its skills yourself. The free plan allows one custom connector.
The steps are at https://qunei.ai/connect#free.

## What this plugin sends, and where

The plugin is instructions and one web address. Nothing in it runs on
your computer. Its one connector is Qunei, at https://qunei.ai/mcp: when you
use it, Claude sends Qunei what you ask it to record or look up,
including the statements and documents you share for that, and only
after you sign in to Qunei. When you ask Claude to read a document you
sent to Qunei, or to save a copy of your books, it downloads the file
from a ten-minute link on https://qunei.ai. It reads or saves the file and
never runs it. Qunei’s privacy policy: https://qunei.ai/privacy.

## Updates

To get Qunei’s updates in Claude, open **Customize**, then **Plugins**, and select **Check for updates** for the Qunei marketplace, or turn on **Sync automatically**.

If you added Qunei with the commands above, get its updates with **claude plugin update qunei@qunei**.

## Help

Email hello@qunei.ai. A person reads every message.

This repository is generated from Qunei’s own source and published by
Qunei. Changes made here are replaced by the next release. See LICENSE
for the terms of use.
