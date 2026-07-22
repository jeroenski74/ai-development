# Claude Code als GitHub Action: opzet en hergebruik

Deze pagina beschrijft hoe Claude Code in deze repository is aangesloten op GitHub,
zodat je dezelfde opzet snel kunt overnemen in andere repository's.

## Wat doet deze integratie?

Eén workflow-bestand, `.github/workflows/claude.yml`, koppelt Claude Code aan twee
manieren om werk te laten oppakken:

1. **`@claude`-mention** — in issue-comments, PR-review-comments en PR-reviews.
   Zodra iemand `@claude` in de tekst zet, start de job `claude-mention` en gaat
   Claude aan de slag met de vraag of instructie uit die comment/review.
2. **Label `claude` op een issue** — fungeert als "toewijzen aan Claude". Zodra dit
   label aan een issue wordt toegevoegd, start de job `claude-label`, die het issue
   leest, de benodigde codewijzigingen maakt en een pull request opent.

Beide triggers staan in hetzelfde workflow-bestand, met eigen `if:`-condities per
job, zodat ze elkaar niet blokkeren en onafhankelijk van elkaar kunnen draaien.

## Vereisten

- De **Claude Code GitHub App** moet op de repository (of organisatie) zijn
  geïnstalleerd.
- Het secret **`CLAUDE_CODE_OAUTH_TOKEN`** moet zijn ingesteld op repository- of
  organisatieniveau (Settings → Secrets and variables → Actions).
- Elke job heeft zijn eigen `permissions`-blok (`contents`, `pull-requests`,
  `issues`, `id-token`) — bewust niet samengevoegd naar workflow-niveau, zodat per
  job duidelijk is welke rechten nodig zijn.

## De workflow

Huidige inhoud van `.github/workflows/claude.yml` in deze repository:

```yaml
name: Claude Code
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  pull_request_review:
    types: [submitted]
  issues:
    types: [labeled]

jobs:
  claude-mention:
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review' && contains(github.event.review.body, '@claude'))
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}

  claude-label:
    if: github.event_name == 'issues' && github.event.label.name == 'claude'
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          prompt: |
            Los issue #${{ github.event.issue.number }} op: lees de beschrijving,
            maak de benodigde codewijzigingen en open een pull request.
```

Toelichting bij een paar keuzes:

- De `actions/checkout@v4`-stap met `fetch-depth: 0` haalt de volledige git-historie
  op vóórdat Claude Code start, zodat Claude toegang heeft tot de repo-inhoud en
  eerdere commits.
- De `claude-label`-job krijgt een eigen `prompt:` mee, zodat Claude direct weet dat
  het issue moet worden opgelost zonder verdere instructie nodig te hebben.
- De `claude-mention`-job krijgt bewust géén vaste `prompt:` mee: de instructie komt
  uit de comment of review zelf.

## Hergebruiken in een andere repository

1. Installeer de Claude Code GitHub App op de nieuwe repository (of organisatie).
2. Voeg het secret `CLAUDE_CODE_OAUTH_TOKEN` toe aan die repository (of hergebruik
   een organisatie-secret).
3. Kopieer `.github/workflows/claude.yml` ongewijzigd naar de nieuwe repository.
4. Voeg eventueel het label `claude` toe aan de repository (Issues → Labels), zodat
   je issues direct aan Claude kunt "toewijzen".
5. Test de opzet: plaats een comment met `@claude` op een issue, of label een issue
   met `claude`, en controleer dat de bijbehorende job start in het tabblad
   **Actions**.

## Aandachtspunten

- Claude Code kan via deze GitHub App **geen bestanden in `.github/workflows/`
  wijzigen** — wijzigingen aan de workflow zelf moeten altijd handmatig (of via een
  PR van een mens) worden doorgevoerd.
- Houd per job een eigen `permissions`-blok aan; voorkom dat rechten breder worden
  dan noodzakelijk voor die specifieke trigger.
- Het secret zelf hoeft nooit in de workflow-inhoud te worden aangepast — alleen de
  naam van het secret wordt gerefereerd.
