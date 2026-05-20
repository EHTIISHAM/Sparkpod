# Contributing to Sparkpod

Thanks for considering a contribution. The bar is low and the welcome is real.

## Quick principles

- **Honest > clever.** If something's a hack, comment it as a hack.
- **No new dependencies without a reason.** Every added requirement is a maintenance tax on everyone.
- **Match the existing style.** Pure prose comments, type hints where they help, clear function names over docstring novels.
- **One change per PR.** Smaller is easier to review and ship.

## Ways to help

### File an issue
Bug reports, weird podcast inputs that confused the ranker, feature requests, documentation gaps. All welcome. Be specific: `clip_NN.json` from the affected source is worth a thousand words.

### Tune the rubric
`config/rubric.json` defines how Sparkpod ranks clip candidates. If you find weights that consistently produce better clips, open a PR with sample outputs.

### Add a frame template
`config/frame_template.json` is one example. Send a PR with your own templates, and we'll work out a multi-template selection mechanism.

### Build a posting integration
Direct YouTube / TikTok / IG / Facebook posting is the biggest open item. If you've worked with those APIs, your help would move this forward significantly.

## Development setup

```bash
git clone https://github.com/EHTIISHAM/sparkpod.git
cd sparkpod
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env               # fill in GEMINI_API_KEY
python ignite_clip.py run --url "<short test podcast URL>" --top 2
```

A "short test podcast" = a 30-45 min episode, so iteration cycles stay fast.

## PR checklist

- [ ] Code runs cleanly with `python -m py_compile core/*.py ignite_clip.py app.py`
- [ ] No new `.env` secrets committed (check `.gitignore` covers your additions)
- [ ] Updated `.env.example` if you added a new config knob
- [ ] Updated relevant section of README if user-facing behavior changed
- [ ] Manual smoke test: ran the pipeline end-to-end on at least one real podcast

## Questions

Open an issue with the `question` label. No question is too basic.
