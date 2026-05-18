# /lovart-video

Generate a video from a text prompt (and optional reference images) using the
Lovart AI agent. Prints the path to a finished video file.

This is a low-level building block — it knows nothing about products, stores,
or social posts. Other skills call it for their video step.

## Usage

```
/lovart-video --prompt "<description>" [--ref <path>]... [--out <path>] [--mode fast|thinking] [--prefer-models '<json>']
```

| Arg | Required | Default | Meaning |
|-----|----------|---------|---------|
| `--prompt` | yes | — | Video description. State duration and aspect ratio here in plain words (Lovart has no flags for them), e.g. "8-second vertical 9:16 clip". |
| `--ref` | no | — | Local image/video reference file. Repeatable. Uploaded to Lovart's CDN, then passed as an attachment. |
| `--out` | no | `/tmp/lovart-video/<timestamp>.mp4` | Destination path for the finished video. |
| `--mode` | no | `fast` | Lovart reasoning depth: `fast` or `thinking`. |
| `--prefer-models` | no | — | JSON soft model preference, e.g. `{"VIDEO":["generate_video_kling_v3"]}`. |

## How it works

The Lovart agent ships as one self-contained Python CLI, `agent_skill.py`, in
the `lovartai/lovart-skill` GitHub repo. The `npx skills add` installer does
not copy that script, so this skill bootstraps it directly into a local cache.

## Workflow

### Step 1 — Preflight

```bash
CACHE="$HOME/.cache/lovart-video"
SKILL="$CACHE/agent_skill.py"
mkdir -p "$CACHE"
if [ ! -f "$SKILL" ]; then
  echo "Fetching Lovart agent_skill.py ..."
  curl -fsSL "https://raw.githubusercontent.com/lovartai/lovart-skill/main/skills/lovart-skill/agent_skill.py" -o "$SKILL" \
    || { echo "ERROR: could not download agent_skill.py from lovartai/lovart-skill."; exit 1; }
fi
set -a
[ -f "$HOME/.openclaw/.env" ] && . "$HOME/.openclaw/.env"
set +a
if [ -z "$LOVART_ACCESS_KEY" ] || [ -z "$LOVART_SECRET_KEY" ]; then
  echo "ERROR: LOVART_ACCESS_KEY / LOVART_SECRET_KEY not set (expected in ~/.openclaw/.env or shell env)."
  exit 1
fi
```

If preflight fails, STOP and tell the user which prerequisite is missing. Do
not attempt generation.

### Step 2 — Upload references

For each `--ref` local file, upload it to Lovart's CDN and capture the URL:

```bash
python3 "$SKILL" upload --file "<ref_path>"
```

`upload` prints JSON `{"url": "<cdn-url>"}` to stdout — read the top-level
`url` key. Collect every URL into the `CDN_URLS` bash array (used in Step 3).
If there are no `--ref` args, skip this step and leave `CDN_URLS` empty.

If an `upload` call exits non-zero or returns no `url`, STOP and report the
error — do not continue to generation.

### Step 3 — Generate

`--attachments` is a single flag that takes one or more URL values. Build the
argument list with a bash array so it is space-safe and absent when there are
no references:

```bash
WORKDIR=$(mktemp -d /tmp/lovart-video.XXXXXX)
ATTACH=()
[ ${#CDN_URLS[@]} -gt 0 ] && ATTACH=(--attachments "${CDN_URLS[@]}")
python3 "$SKILL" chat --prompt "$PROMPT" --mode "$MODE" "${ATTACH[@]}" --json --download --output-dir "$WORKDIR"
```

`$PROMPT` is the caller's `--prompt` text and `$MODE` is the resolved
`--mode` value. `ATTACH` expands to one `--attachments` flag followed by every
URL in `CDN_URLS`, or to nothing when `CDN_URLS` is empty. Add
`--prefer-models '<json>'` only if the caller passed it. `chat` sends the
prompt, waits for completion, and downloads artifacts into `--output-dir`.

### Step 4 — Confirm high-cost operations

If the Step 3 JSON has `final_status` equal to `"pending_confirmation"`,
Lovart is waiting for cost approval:

1. Report the cost figure from the response to the user.
2. Confirm (the user invoked `/lovart-video` intentionally):
   ```bash
   python3 "$SKILL" confirm --thread-id "<thread_id>" --json
   ```
   Inspect the `confirm` response JSON; if it indicates failure, STOP and
   report it rather than calling `result`.
3. Retrieve the finished result:
   ```bash
   python3 "$SKILL" result --thread-id "<thread_id>" --json --download --output-dir "$WORKDIR"
   ```

### Step 5 — Resolve the output file

From the final JSON payload:
- Verify `generation_succeeded` is `true`. If not, print the error and exit
  non-zero — never print a fake path.
- Set `DOWNLOADED` to the first downloaded video artifact's
  `downloaded[0].local_path`.
- Set `OUT` to the caller's resolved `--out` value, defaulting to
  `/tmp/lovart-video/<timestamp>.mp4` when `--out` was not given.

```bash
mkdir -p "$(dirname "$OUT")"
mv "$DOWNLOADED" "$OUT"
echo "$OUT"
```

Print the absolute output path (`$OUT`) on the LAST line of your response so
callers can capture it.

## Notes

- Duration and aspect ratio must be in the `--prompt` text — there are no flags.
- Credentials live in `~/.openclaw/.env` (`LOVART_ACCESS_KEY`, `LOVART_SECRET_KEY`).
- `agent_skill.py` is cached at `~/.cache/lovart-video/`; delete it to force a refresh.
- Each `chat` call may incur Lovart cost; the `confirm` gate surfaces the figure.
