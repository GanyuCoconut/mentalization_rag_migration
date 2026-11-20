# Response Guidance v2 Construction

This pipeline builds a candidate-only, evidence-grounded mentalization corpus.
It does not overwrite `mentalization_guidance.db` and does not enable model
output for runtime use before human review.

## 1. Required sources

The source manifest is `source_manifest.json`. The first release requires:

1. *Mentalization-Based Treatment for Personality Disorders: A Practical Guide*.
2. *心智化：依附关系、情感调节、自我发展*.
3. Rossouw, Wiwe, and Vrouva (eds.), *Mentalization-Based Treatment for
   Adolescents: A Practical Treatment Guide*, Routledge, 2021, 1st edition,
   DOI `10.4324/9780429323928`.

Put the lawfully acquired MBT-A PDF at:

```text
/workspace/ references/books/Mentalization-Based-Treatment-for-Adolescents-a-Practical-Trudie-Rossouw-Maria-Wiwe-Ioanna-Vrouva-.pdf
```

Do not substitute a preview, web summary, or unofficial copy.

## 2. Configure the external model

Create a local, unversioned environment file based on `api.env.example`:

```bash
cd /workspace/teen_mh_agent
cp knowledge_sources/response_guidance_v2/api.env.example .env.guidance.local
chmod 600 .env.guidance.local
```

Fill in:

```text
GUIDANCE_EXTRACTION_API_BASE
GUIDANCE_EXTRACTION_API_KEY
GUIDANCE_EXTRACTION_MODEL
```

The endpoint must implement OpenAI-compatible `chat/completions`. Keep
`GUIDANCE_EXTRACTION_JSON_MODE=json_schema` when supported; otherwise use
`json_object` or `prompt_only`.

## 3. Run in tmux

```bash
enter-wyh
tmux new -s guidance-v2
cd /workspace/teen_mh_agent
set -a
source .env.guidance.local
set +a
```

Check all required PDFs before spending API tokens:

```bash
PYTHONPATH=src /opt/conda/envs/pycho/bin/python -m teen_mh_agent.guidance_pipeline status
```

Parse the books into page-anchored extraction chunks:

```bash
PYTHONPATH=src /opt/conda/envs/pycho/bin/python -m teen_mh_agent.guidance_pipeline prepare
```

Run a two-chunk API pilot and inspect the output:

```bash
PYTHONPATH=src /opt/conda/envs/pycho/bin/python -m teen_mh_agent.guidance_pipeline \
  extract-atoms --max-chunks 2

sed -n '1,2p' knowledge_work/response_guidance_v2/atoms.jsonl
```

If the pilot is source-faithful, resume the full run. Completed chunk IDs are
automatically skipped:

```bash
mkdir -p logs
set -o pipefail
PYTHONPATH=src /opt/conda/envs/pycho/bin/python \
  -m teen_mh_agent.guidance_pipeline extract-atoms \
  2>&1 | tee -a logs/guidance_extract_atoms.log

PYTHONPATH=src /opt/conda/envs/pycho/bin/python \
  -m teen_mh_agent.guidance_pipeline revalidate-atoms
PYTHONPATH=src /opt/conda/envs/pycho/bin/python \
  -m teen_mh_agent.guidance_pipeline clean-atoms
PYTHONPATH=src /opt/conda/envs/pycho/bin/python \
  -m teen_mh_agent.guidance_pipeline status
```

Detach from tmux with `Ctrl-b`, then `d`. Reattach with
`tmux attach -t guidance-v2`. If extraction stops, run the same command again;
records already written to `atoms.jsonl` are skipped by chunk ID.

`clean-atoms` preserves raw `atoms.jsonl` and writes:

- `atoms_cleaned.jsonl`: normalized candidate atoms used by synthesis and DB build.
- `atom_cleaning_report.json`: counts for index removal, evidence-length gates,
  exact duplicates, population normalization, and near-duplicate review groups.

Spot-check the cleaned atoms and audit report before synthesis. Then run:

```bash
PYTHONPATH=src /opt/conda/envs/pycho/bin/python \
  -m teen_mh_agent.guidance_pipeline build-guidance-packets

# Pilot: synthesize two controlled topics, then verify every card statement.
PYTHONPATH=src /opt/conda/envs/pycho/bin/python \
  -m teen_mh_agent.guidance_pipeline synthesize --max-packets 2
PYTHONPATH=src /opt/conda/envs/pycho/bin/python \
  -m teen_mh_agent.guidance_pipeline verify-guidance --max-cards 10
```

Inspect these files before a full run:

- `guidance_packet_report.json`: topic coverage and source quotas.
- `guidance_candidates.jsonl`: candidate cards and locally rejected cards.
- `guidance_verification.jsonl`: statement-level support decisions and risk flags.

Cards must pass deterministic evidence-map, population, crisis-isolation, and
unsupported-generic-action gates. The verifier is an additional semantic
pre-screen, not a replacement for professional review. Packets that produce no
accepted card remain retryable.

After the pilot is satisfactory, resume all remaining topics and verify them:

```bash
PYTHONPATH=src /opt/conda/envs/pycho/bin/python \
  -m teen_mh_agent.guidance_pipeline synthesize
PYTHONPATH=src /opt/conda/envs/pycho/bin/python \
  -m teen_mh_agent.guidance_pipeline verify-guidance
```

Apply audited manual revisions, attach controlled use scopes, reverify only the
revised cards, and freeze the machine-screened candidate set:

```bash
PYTHONPATH=src /opt/conda/envs/pycho/bin/python \
  -m teen_mh_agent.guidance_pipeline curate-guidance

PYTHONPATH=src /opt/conda/envs/pycho/bin/python \
  -m teen_mh_agent.guidance_pipeline verify-guidance \
  --cards knowledge_work/response_guidance_v2/guidance_candidates_curated.jsonl \
  --output knowledge_work/response_guidance_v2/guidance_verification_curated.jsonl

PYTHONPATH=src /opt/conda/envs/pycho/bin/python \
  -m teen_mh_agent.guidance_pipeline finalize-guidance
```

The final candidate file is `guidance_candidates_final.jsonl`. Its scopes are:

- `direct_support`: eligible for ordinary retrieval only after human approval.
- `clinician_planning`: review material for trained clinicians, never direct response text.
- `family_session`: requires a facilitated family-session context.
- `professional_network`: for professional collaboration or supervision.
- `crisis_only`: review-only safety material; never returned by ordinary runtime search.

`runtime_eligible=true` is valid only for `direct_support`. The crisis card is
always `runtime_eligible=false`; it cannot authorize an AI system to decide
hospitalization or replace a fixed crisis-escalation protocol.

Build the candidate-only database:

```bash
PYTHONPATH=src /opt/conda/envs/pycho/bin/python -m teen_mh_agent.guidance_pipeline build-db
PYTHONPATH=src /opt/conda/envs/pycho/bin/python -m teen_mh_agent.guidance_pipeline status
```

The first build refuses to overwrite an existing v2 database. Use `--replace`
only after manually preserving the prior database and work JSONL files.

## 4. Demo prototype mode

`demo.py` enables the machine-screened candidate v2 database by default because
the Demo is explicitly a research prototype. In this mode RetrieverAgent uses
only scoped v2 `direct_support` cards and does not mix v1 guidance or raw book
chunks back into the response-guidance result.

```bash
/opt/conda/envs/pycho/bin/python demo.py --model qwen3-14b --port 8002
```

Use `--disable-guidance-v2` for an ablation or rollback run. Library/runtime
callers remain approved-only by default; candidate retrieval requires the
explicit environment variable `TEEN_MH_GUIDANCE_V2_PROTOTYPE=1`. Any unmasked
high-risk phrase makes ordinary guidance retrieval return no cards so the fixed
safety escalation path remains authoritative.

## 5. Human review

Export cards together with evidence atoms, source title, PDF pages, and the
verbatim evidence excerpt:

```bash
PYTHONPATH=src /opt/conda/envs/pycho/bin/python -m teen_mh_agent.guidance_pipeline export-review
```

Create a JSONL decision file following `review_decisions.example.jsonl`.
Evidence atoms must first become `evidence_checked`; a card cannot become
`approved` while any linked atom remains candidate or rejected.

```bash
PYTHONPATH=src /opt/conda/envs/pycho/bin/python -m teen_mh_agent.guidance_pipeline \
  apply-review --decisions /path/to/review_decisions.jsonl
```

All review actions are recorded in the `review_events` table. Runtime search
returns only approved, runtime-eligible `direct_support` cards. Other scopes
remain unavailable even if their content is approved for research use.

## 6. Future sources

Add a source to `source_manifest.json` with one of these roles:

```text
core_theory
adolescent_adaptation
quality_and_safety
case_reference
```

New sources default to disabled candidate sources. The v2 schema already
reserves a separate `case_references` table so later case material cannot be
mixed into the core guidance policy.
