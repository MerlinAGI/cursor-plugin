---
name: listen-research
description: >-
  Run real user research from Cursor via the Listen Labs MCP server: create,
  edit, launch, and analyze AI-moderated user-interview studies. Use this
  whenever the user wants to launch a study, interview users or customers,
  recruit participants or a panel, validate an idea, test messaging / pricing /
  concepts / prototypes with real people, add screener questions, or pull
  findings, themes, quotes, or transcripts from an existing study, even if
  they never say Listen Labs. Also use it when the user asks to connect
  Listen Labs to Cursor or asks what their Listen MCP connection can do.
---

# Listen Labs Research

Listen Labs runs AI-moderated user interviews: describe a research goal, a study gets
built (interview guide + screener + recruitment), real respondents are recruited and
interviewed, and an analysis report is generated from the transcripts. Everything below
runs through the Listen Labs MCP server.

## Step 0: check the connection

You need the Listen MCP tools: `create_study`, `edit_study`, `launch_study`,
`publish_study`, `get_study_state`, `list_studies`, `list_creatable_orgs`,
`get_study_analysis`, `get_study_responses`, `get_response`, `search_across_studies`,
`move_study`, `manage_folder`. They may sit under an MCP server prefix, and their
schemas may need to be loaded before first use.

If none are available, help the user connect, then continue with their request:

- **Cursor Marketplace:** Customize → search "Listen Labs" → Install → Connect, then
  sign in with OAuth.
- **Custom MCP:** Settings → Tools & MCP → Add Custom MCP, then add:

```json
{
  "mcpServers": {
    "listen-labs": {
      "url": "https://mcp.listenlabs.ai/mcp"
    }
  }
}
```

- **EU:** use `https://mcp.eu.listenlabs.com/mcp` instead.
- Docs: https://docs.listenlabs.com/mcp-docs/connect

The user needs a Listen Labs account with an organization. Verify the connection with
`list_creatable_orgs`. A successful reply also shows which workspace(s) they can use.

## Using the tools

The MCP tool descriptions are the authoritative reference. They document every
parameter, pagination rule, and output requirement. Read a tool's schema before calling
it. Parameter casing varies by tool (`studyId` vs `study_id`), and `get_response` needs
a `readable_id` that comes from `get_study_responses`.

| Tool                    | What it's for                                                    |
| ----------------------- | ---------------------------------------------------------------- |
| `list_creatable_orgs`   | Workspaces the user can create studies in                        |
| `list_studies`          | Find studies. Pass `textHint` when the user names one            |
| `search_across_studies` | Keyword search across studies                                    |
| `create_study`          | Seed a new study and start the guided setup conversation         |
| `edit_study`            | Continue setup, or edit any study in natural language            |
| `get_study_state`       | Snapshot: guide, screener, recruitment, launch readiness + costs |
| `publish_study`         | Make edits visible to respondents. Does not start recruitment    |
| `launch_study`          | Publish + start recruitment. Spends credits                      |
| `get_study_analysis`    | AI analysis report, once available                               |
| `get_study_responses`   | Interview transcripts, paginated and filterable                  |
| `get_response`          | Deep-dive on one respondent                                      |
| `move_study`            | File a study into an existing folder, or back to the dashboard   |
| `manage_folder`         | Create, rename, re-nest, or delete a folder                      |

## Workflow 1: create a study

1. Make sure you know what the user wants to learn, who to talk to, and roughly how many
   people. If they already told you, confirm in one line. Do not re-interview them.
2. Call `list_creatable_orgs`. One org: proceed. Several: ask which, and pass
   `orgName` to `create_study`. Never guess.
3. Call `create_study` with a rich prompt (goal, audience, what to learn). Rich seeds
   skip setup turns.
4. Keep the returned `studyId` and `chatId` and continue every setup turn through
   `edit_study` with both. Answer the setup agent's structured choices with the
   `nextActions` buttons echoed by the previous turn: copy them verbatim, do not
   hand-craft them. If a button choice does not take (the agent re-asks and nothing
   changed), state the choice as a plain `prompt` instead. Relay real decisions (panel
   vs own participants, interview format, target size) to the user; answer mechanical
   steps yourself when the user already gave you the answer.
5. After every turn, show the entire study guide verbatim: every section, question,
   and answer option, in order. Never summarize. The user is signing off on exactly what
   respondents will see. If the guide is still empty, say so.
6. Iterate with `edit_study` until they are happy. Creating is not launching. "Set up a
   study" means draft it.

For website / app / prototype tests, pick a screen-share interview format during setup
(respondents share their screen while completing tasks) and make sure the URL and
concrete, ordered tasks appear in the guide respondents see.

## Workflow 2: launch (spends real money, always gate)

1. Call `get_study_state` and read the launch info: costs, credit balance, what will
   launch and what would be skipped, any blockers.
2. Show the user the cost and balance impact, and get an explicit go-ahead. Never
   launch on your own initiative. Only an unambiguous "launch it / go live / start
   recruiting" counts.
3. Call `launch_study`, then report what launched, what was skipped and why, and the new
   balance. If credits are short, the user adds them in the Listen dashboard. That
   cannot be done over MCP. Re-calling is safe: already-running recruitment is not
   restarted.

Panel recruitment (Listen finds participants) has an upfront cost. A self-recruit link
(user brings participants) bills per response.

## Workflow 3: edit an existing study

1. Find it with `list_studies` + `textHint`. Ask the user to pick if several match.
2. Call `get_study_state` so you and the user see what's there now.
3. Send a plain-language `edit_study` prompt. Omit `chatId` on the first turn; reuse the
   echoed one for follow-ups.
4. Show the updated guide verbatim, same rule as creation.
5. If the study is already live, edits are not visible to respondents until the study
   is re-published. Tell the user, and call `publish_study` when they confirm. Use
   `launch_study` instead only when they also want to start new recruitment.

## Workflow 4: read results

- `list_studies` shows status, response counts, and whether analysis is ready.
- `get_study_analysis` for the report; `get_study_responses` for transcripts. Filter by
  question numbers or respondents instead of pulling everything. `get_response` to zoom
  in on one person. `search_across_studies` for "did we ever research X?".
- Grounding is required: every quote you surface must be verbatim and immediately
  followed by its exact `[Source]` link copied from the tool output. For `get_response`,
  build the link from the item's `source_url`. Never paraphrase, merge, or re-attribute
  quotes. If a claim is not in the returned content, do not make it. If the data does
  not answer the question, say so rather than stretching, and offer a follow-up study.

## Ground rules

- Track `studyId` and the latest `chatId` across turns. Each `edit_study` call takes
  either a prompt or a button click, never both.
- Never re-call `create_study` to fix a study you just made. That creates a second
  study. Edit the one you have.
- Call `get_study_state` before mutating any study you did not just create.

## Worked examples

Compressed flows showing tool order and where to stop and talk to the user. Calls are
shown as `tool(args)`; responses trimmed to the fields that drive the next step.

### Create, then launch

> "I want to find out if marketing managers would pay for an AI ad-copy tool. Set up
> interviews with ~25 of them and get it running."

"Get it running" is launch intent. Still confirm cost before spending.

```
list_creatable_orgs()              → one org: proceed silently
create_study(prompt: "Interview ~25 marketing managers who run paid campaigns.
  Goal: would they pay for AI-written ad copy: current workflow, pain points,
  price expectations.")
                                   → studyId, chatId,
                                     nextActions: [use-panel, bring-my-own]
```

Render the guide verbatim, then relay the real decision: panel or their own contacts?
User says panel:

```
edit_study(studyId, chatId, buttonClick: <the use-panel button, copied from nextActions>)
…continue the stages the same way; answer with a prompt where the user
already told you (audience, ~25 people)…
get_study_state(studyId)           → cost, balance, blockers
```

Show the cost and new balance, then ask to confirm. Only on a clear yes:
`launch_study(studyId)`. Report what launched and the new balance.

### "Set up a study" (create is not launch)

> "Set up a study to test our new onboarding flow with existing users. I'll send it to
> our mailing list myself."

Same creation flow, choosing self-recruit ("bring my own"). Render the final guide and
stop. Tell them it is drafted, not live. Do not call `launch_study` until they answer.
Launching returns the link to share and bills per response.

### Edit a live study, then publish

> "On my trust study, add a screener question for people who've used a face-rating app
> before, and make sure new respondents see it."

```
list_studies(textHint: "trust")    → 2 matches: ask which one
get_study_state(studyId)           → live, published, recruitment running
edit_study(studyId, prompt: "Add a screener question filtering for people who have
           used a face-rating app before; screen out the rest.")
```

Show the updated screener verbatim. Respondents still see the old version until the
study is re-published. Say so, and on confirm: `publish_study(studyId)`. If they also
want more respondents, that is `launch_study` with the usual cost gate.

### Findings with sourced quotes

> "What did people say about pricing in the ad message study? Give me the highlights."

```
list_studies(textHint: "ad message")  → 1 match, has_analysis: true
get_study_analysis(study_id)
```

Report the themes. Every quote verbatim with its `[Source]` link exactly as returned.

If the analysis does not cover the question, go to transcripts (filtered, not wholesale):
`get_study_responses(study_id, question_numbers: [4, 5])`, or `get_response(study_id,
readable_id: 7)` to zoom in on one person. If the study never asked about it, say so
and offer a follow-up study instead of stretching.

### Website usability test (screen share)

> "We just redesigned checkout at shop.acme.com. Can you set up a test where ~15 people
> go through the site and think aloud?"

Two things matter here: a screen-share interview format, and a task-based guide that
names the URL. Seed both in `create_study`. At the interview-format stage, pick a
screen-share mode from `nextActions` (voice + screen is the usual pick). Before
sign-off, check the guide: the URL appears in the respondent-facing instructions, tasks
are concrete and ordered, and there is think-aloud prompting. Launch gate unchanged.

### Not connected yet

> "Can you launch a Listen study for me?" but no Listen tools exist in the session.

Give the Step 0 setup, have them connect, verify with `list_creatable_orgs`, then
proceed as above.
