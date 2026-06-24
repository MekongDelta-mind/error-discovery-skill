# Review Loop

This is the ongoing interactive phase. The app is running, the human is reviewing. You are an active participant.

## The two loops: breadth and depth

Error discovery alternates between two modes:

**Breadth**: cover as much of the dataset as possible. Diverse samples, random picks, fill cluster gaps. The goal is finding *different* failure modes.

**Depth**: once you find a mode, go deep. Scan all records for instances, re-review earlier items, refine the definition. The goal is understanding *one* failure mode thoroughly.

The outer loop alternates: go broad until you find something, then go deep on it, then go broad again to find the next thing.

## Launch and monitor

1. Start the Python server in the background.
2. Open the app in the browser.
3. Tell the user the app is ready. Explain the interaction briefly.
4. **Monitor the annotations file** using the Monitor tool. React when new annotations come in.

## Process annotations as they arrive

When new annotations appear:
1. Read all annotations.
2. Categorize each into a failure mode — match to existing or create new.
3. Maintain a running taxonomy: `{mode_name: {description, count, example_ids, example_quotes[]}}`.
4. Push the updated taxonomy to `POST /api/patterns`.
5. Track which records have been reviewed and which clusters/dimensions are covered.

## Go deep: scan for failure mode instances

This is the depth loop. When a new failure mode is discovered (or an existing one gets clearer), scan **all** records for instances — not just unreviewed ones.

**Spawn one background subagent per failure mode.** The subagent gets the mode name, description, and example quotes, reads through all records, and returns a list of suggested annotations: `{record_id, text, start, end}`. One mode = one subagent, not one per record. Multiple modes run in parallel. This happens in the background while the human keeps reviewing.

When subagents return, merge their suggestions and push to `POST /api/suggestions`. The UI picks them up automatically and shows them in the Progress view queue and as purple highlights in the article view.

Suggestions land in two buckets:
1. **Already-reviewed records**: the reviewer may have missed this because the mode wasn't in their head yet (criteria drift).
2. **Unreviewed records**: new coverage. Add them to the sample if they aren't already.

Agent suggestions are never ground truth. The human accepts or dismisses them. Lean toward recall over precision — the cost of a false positive (dismissed) is low, the cost of a miss is high.

## Go broad: propose new samples

This is the breadth loop. After the human reviews a batch:
1. **Cover gaps**: sample from unreviewed clusters, topics, or feature regions.
2. **Random exploration**: always include a few random picks.
3. Push new samples to `POST /api/samples`. The app shows a banner.
4. Tell the human what was added and why.

## Encourage re-review

Don't treat review as one-shot. The reviewer's criteria shift as they see more data (criteria drift).

The agent handles part of this automatically: background subagents scan already-reviewed records and push suggestions. But the agent can't catch everything. After the reviewer has gone through a batch and discovered new modes, explicitly encourage them to re-read earlier records with fresh eyes. Say something like: "You've found 3 new failure modes since you reviewed records 0-5. Worth a second pass — you'll likely spot things you missed the first time."

## Report and converge

Periodically:
- Report the failure mode taxonomy with confirmed and suggested counts.
- Report coverage: records reviewed, clusters/dimensions covered, what remains.
- Track convergence: are new records revealing new modes, or mostly repeats?
- When discovery rate drops, suggest stopping or narrowing focus.

## Key principles

1. **The human notices, the agent organizes.** Free-text notes become structured taxonomy. Don't make the human categorize.
2. **The agent drives coverage.** The agent proposes new samples based on what's been found and what's missing.
3. **Live feedback loop.** The agent monitors and reacts as annotations come in.
4. **Guard against blind spots.** Always include random samples alongside cluster-based ones.
5. **Iterate, don't one-shot.** Criteria drift is real. Multiple passes over the same data surface different things.
6. **Agent suggests, human confirms.** Suggestions are visually distinct and require accept/dismiss. Favor recall over precision.
7. **Breadth then depth, repeat.** Go broad to find modes, go deep to understand them, go broad again.
