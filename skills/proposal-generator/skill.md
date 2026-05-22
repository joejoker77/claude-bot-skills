# Skill: proposal-generator

Generate a branded AAS proposal and publish live to proposals.ai-automation.studio.

## Trigger
"proposal", "оффер", "КП", "new proposal", "/proposal-generator"

## Steps

1. Ask user to paste client notes (or describe the client situation)
2. Run: `python3 ~/aas-proposals/generate.py "[client text]"`
3. Report back:

```
✓ Live:  https://proposals.ai-automation.studio/[slug]
✓ Code:  AAS-XXXX
Send URL and code separately to the client.
```

## Notes
- CF_TOKEN must be set (lives in `~/.env.aas`, mode 600). If missing, ask Boris.
- Each run produces a brand-new URL and a fresh AAS-XXXX code. To update an existing proposal, run again — the new link replaces the old one for that client.
- For shallow one-pagers a short brief is fine. For deep client-specific pages (like the existing `ai-assistant.gg/sedov`), feed the generator the FULL brief, not a one-liner, otherwise the output will be thin.
