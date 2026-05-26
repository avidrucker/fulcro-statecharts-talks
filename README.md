# fulcro-statecharts-talks

Captured talks by [Tony Kay](https://github.com/awkay) on the
[Fulcrologic statecharts library](https://github.com/fulcrologic/statecharts),
distilled into executive summaries + key-takeaways indices for personal
study and citation.

Sister project to [`warmed-skills`](https://github.com/avidrucker/warmed-skills)
(captures Casey Muratori's "Where Does Bad Code Come From?" talk) and
[`yegor-pm-skills`](https://github.com/avidrucker/yegor-pm-skills)
(captures Yegor Bugayenko's XDSD talk): same shape — one subdir per
talk, with both an executive summary (TL;DR + Salient Points + Outline)
and a key-takeaways index (timestamped emoji bullets). Applied here to
Tony Kay's Fulcro statecharts content.

The pipeline that produced the transcripts and summaries lives in the
sister repo [`talk-distill-skills`](https://github.com/avidrucker/talk-distill-skills).
This repo holds the *outputs*; that one holds the *pipeline*.

## What's captured

| Talk | Length | YouTube | Subdir |
|---|---|---|---|
| **Statecharts** (basics walkthrough — Q&A-driven team talk) | 1h 16m | [`RMONaWbgnA8`](https://www.youtube.com/watch?v=RMONaWbgnA8) | [`fulcro_statecharts_talk/`](fulcro_statecharts_talk/) |
| **Statecharts Marathon** (advanced topics — assumes basics, deep dive on SCXML semantics + async parking processor) | 3h 28m | [`06-DMDXxSDM`](https://www.youtube.com/watch?v=06-DMDXxSDM) | [`fulcro_statecharts_marathon/`](fulcro_statecharts_marathon/) |

Each subdir has:

- **`executive_summary.md`** — TL;DR + Salient Points + Outline. First-read friendly. (File shape originated in `warmed-skills/bad_code_talk/talk_summary.md`.)
- **`key_takeaways.md`** — timestamped bullet index. Re-navigation friendly, jump-into-the-video friendly. (File shape originated in `yegor-pm-skills/XDSD_YouTube_Talk/key_takeaways.md`.)
- **`captions.SRT`** (`talks` branch only) — raw `en-orig` auto-captions from YouTube.
- **`transcript.txt`** (`talks` branch only) — light-edit, near-literal, speaker-labelled (`Tony Kay:` / `Host:`) paragraph transcript produced by [`talk-distill-skills`](https://github.com/avidrucker/talk-distill-skills).

## Branch layout (main vs. talks)

This repo uses two long-lived branches so a default clone stays lean:

| Branch | Contents |
|---|---|
| `main` | README, LICENSE, .gitignore, and each subdir's `executive_summary.md` + `key_takeaways.md`. Bytes count: tens of KB. |
| `talks` | Everything on `main` **plus** `captions.SRT` + `transcript.txt` in each subdir. Bytes count: ~1.2 MB. |

The summaries are hand-curated and small, so they live on `main` and are
versioned with the rest of the repo. The SRTs and transcripts are bulky
and trivially regenerable from YouTube via
[`yt-dlp`](https://github.com/yt-dlp/yt-dlp) — so they live on a side
branch to keep clones lean while still making the cite-and-quote source
material one branch-switch away.

### Clone just the lean main branch

```bash
git clone --single-branch --branch main https://github.com/avidrucker/fulcro-statecharts-talks.git
```

### Clone and check out the talks branch instead

```bash
git clone https://github.com/avidrucker/fulcro-statecharts-talks.git
cd fulcro-statecharts-talks
git checkout talks
```

### Already cloned? Switch between them

```bash
git fetch origin
git checkout talks   # captions.SRT + transcript.txt appear
git checkout main    # captions.SRT + transcript.txt disappear
```

Git itself manages the raw files appearing/disappearing as you switch.

## Reading order

If you're new to statecharts in general, read
[`fulcro_statecharts_talk/executive_summary.md`](fulcro_statecharts_talk/executive_summary.md)
first — it explains the formalism, the protocol architecture, and the
hard-won SQL-transaction design for the backend's exactly-once event
delivery.

Then read
[`fulcro_statecharts_marathon/executive_summary.md`](fulcro_statecharts_marathon/executive_summary.md)
for the deep semantic dive: enabled-transition algorithm, conflict
resolution, history nodes, invocations, local-data discipline, and the
**async parking processor** Tony added recently — a major engine swap
that landed without breaking any existing chart, which validates the
protocol architecture in retrospect.

## Reproducing the distillation

The transcripts and summaries here weren't hand-typed — they came out of
[`talk-distill-skills`](https://github.com/avidrucker/talk-distill-skills)'s
pipeline. To regenerate (or to apply it to a third Tony Kay talk):

```bash
# In a clone of talk-distill-skills:
yt-dlp --skip-download --write-auto-subs --sub-langs en-orig \
       --convert-subs srt \
       --output "captions.%(ext)s" \
       "https://www.youtube.com/watch?v=<VIDEO_ID>"
python3 scripts/srt_to_transcript.py captions.srt transcript.txt \
        --speakers "Tony Kay,Host" \
        --preamble-skip 27
```

The summaries themselves are written by reading the transcript closely —
not (yet) automated. See the
[`talk-summarize` / `talk-takeaways`](https://github.com/avidrucker/talk-distill-skills#current-state)
sub-skills (planned) for that step.

## License

MIT for this repo's structure, scripts, and curated summaries — see
[LICENSE](LICENSE).

Talk content (the captions and transcript on the `talks` branch, and the
summaries which paraphrase the talks closely) is derivative of Tony Kay's
recordings and remains his intellectual property. It is included here for
personal study and citation, not redistribution. If you want to share or
re-host any of this, ask Tony first.

## Credits

- All talk content © [Tony Kay](https://github.com/awkay) /
  [Fulcrologic](https://github.com/fulcrologic).
- Distillation pipeline:
  [`talk-distill-skills`](https://github.com/avidrucker/talk-distill-skills).
