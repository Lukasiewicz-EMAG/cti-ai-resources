# CTI-RelEx: data and prompts for STIX 2.1 relationship extraction

This directory contains the evaluation data and the prompts used in the paper:

> **CTI-RelEx: Automated STIX Relationship Extraction Using Large Language Models**

CTI-RelEx is a single-stage approach in which a large language model (LLM) jointly detects and types STIX 2.1 relationships between entities already extracted from a cyber threat intelligence (CTI) report. Candidate entity pairs are restricted by an analyst-curated STIX compatibility matrix and processed in batches within five prompt groups, each with its own decision rules. Every predicted relationship is justified with a quote from the report. An optional LLM-as-a-judge step based on G-Eval assigns a confidence score to each prediction.

## Contents

```
stix-relations/
├── data/                                   # 9 expert-annotated report fragments
│   ├── rep_2a740ecf.json
│   ├── ...
│   └── rep_eae8e825.json
└── prompts/
    ├── groups_v3_cot/                      # single-stage group prompts, no justification field
    │   ├── S1_obs.yaml
    │   ├── S2_host.yaml
    │   ├── S3_payload.yaml
    │   ├── S4_infra.yaml
    │   └── S5_prose.yaml
    ├── groups_v3_cot_justification/        # the same prompts, with a quote-based justification per relationship
    │   └── S1_obs.yaml ... S5_prose.yaml
    ├── azerg_prompts_paper.yaml            # AZERG baseline prompts (T3, T4)
    ├── geval_confidence_score.yaml         # G-Eval "Correctness" metric for the judge model
    └── geval_confidence_score_no_think.yaml# the same metric with the Qwen3 /no_think switch
```

## Source reports

The evaluation set was built from two publicly available CTI reports. They come from different sources, describe different domains (industrial control systems vs. enterprise intrusion and ransomware) and require different depths of technical expertise. Each report was manually divided into contextually complete fragments.

| Source report | Publisher | Fragments | Relationships |
|---|---|---|---|
| [INCONTROLLER: New State-Sponsored Cyber Attack Tools Target Multiple Industrial Control Systems](https://cloud.google.com/blog/topics/threat-intelligence/incontroller-state-sponsored-ics-tool) (13 April 2022) | Mandiant (Google Cloud Threat Intelligence) | `rep_3a3a69f8`, `rep_7cf37921`, `rep_aa055e82`, `rep_aa71416d` | 69 |
| [Flash Alert: EtherRat and TukTuk C2 End in The Gentleman Ransomware](https://thedfirreport.com/2026/05/11/flash-alert-etherrat-and-tuktuk-c2-end-in-the-gentleman-ransomware/) (11 May 2026) | The DFIR Report | `rep_2a740ecf`, `rep_3ded5086`, `rep_687c7882`, `rep_9691ae93`, `rep_eae8e825` | 174 |

The report text in the data files is reproduced from these publications for research purposes. All rights to the original text remain with their publishers.

## Data

### Annotation procedure

For each fragment, an expert CTI analyst identified the entities in the text. The entity list was passed to the candidate generator, which produced every ordered entity pair permitted by the STIX compatibility matrix. The analyst then annotated each candidate: either assigning the relationship type supported by the text or rejecting the pair. Annotation was iterative. Whenever the text implied a relationship that no candidate covered, the entity list was updated, the candidates were regenerated and the new set was annotated again.

### Statistics

| | Value |
|---|---:|
| Report fragments | 9 |
| Words (title + text) | 2,949 |
| Entities (excluding the report object) | 190 |
| Candidate pairs permitted by the matrix | 1,916 |
| Annotated relationships | 243 |

Distribution of relationship types:

| Relationship type | Count | Share |
|---|---:|---:|
| `uses` | 211 | 0.87 |
| `targets` | 13 | 0.05 |
| `compromises` | 11 | 0.05 |
| `drops` | 7 | 0.03 |
| `loads` | 1 | 0.00 |

Entity types: `infrastructure` (48), `attack-pattern` (34), `tool` (27), `file` (26), `url` (18), `malware` (14), `domain-name` (8), `intrusion-set` (4), `threat-actor` (4), `process` (4), `vulnerability` (2), `windows-registry-key` (1).

Per fragment:

| Fragment | Source | Words | Entities | Relationships |
|---|---|---:|---:|---:|
| `rep_2a740ecf` | The DFIR Report | 177 | 21 | 60 |
| `rep_3a3a69f8` | Mandiant | 271 | 4 | 5 |
| `rep_3ded5086` | The DFIR Report | 204 | 30 | 42 |
| `rep_687c7882` | The DFIR Report | 301 | 15 | 23 |
| `rep_7cf37921` | Mandiant | 217 | 9 | 27 |
| `rep_9691ae93` | The DFIR Report | 336 | 33 | 21 |
| `rep_aa055e82` | Mandiant | 104 | 8 | 10 |
| `rep_aa71416d` | Mandiant | 233 | 9 | 27 |
| `rep_eae8e825` | The DFIR Report | 1,106 | 61 | 28 |

### File format

Each file in `data/` is one report fragment. The relevant fields are:

- `id`: fragment identifier (`rep_...`);
- `title` and `description`: the fragment text. The title holds the first sentence and the description holds the rest, so the full text is `title + description`;
- `entities_v2`: a list holding the report object, the entities and the relationships;
- `entity_counts_v2`: the number of objects of each type.

An entity has an identifier, a title and a STIX type:

```json
{"id": "ent_0dafb2e6", "data": {"title": "S3 buckets (payload hosting)", "type": "infrastructure"}}
```

A relationship (`"type": "relation"`) links two entities. The `key` field holds the STIX relationship type:

```json
{
  "id": "a25506321e78",
  "data": {
    "source": "ent_177229e8", "source_type": "infrastructure",
    "target": "ent_d1056fb9", "target_type": "infrastructure",
    "key": "uses",
    "type": "relation"
  },
  "metadata": {"report_id": "rep_2a740ecf"}
}
```

## Prompts

### Single-stage group prompts (`groups_v3_cot*`)

Each candidate pair goes to the first applicable of five prompt groups. One prompt per group handles all of the group's candidates for a fragment in a single model call.

| File | Group | Candidates covered | Relationship types |
|---|---|---|---|
| `S1_obs.yaml` | OBS | one endpoint is a network observable (`url`, `domain-name`, `ipv4-addr`, ...) | `communicates-with`, `consists-of` |
| `S2_host.yaml` | HOST | pairs with `file`, `process` or `windows-registry-key` | `downloads`, `drops`, `loads`, `spawns`, `targets`, `uses` |
| `S3_payload.yaml` | PAYLOAD | malware → malware and malware ↔ tool | `downloads`, `drops`, `loads`, `uses`, `variant-of` |
| `S4_infra.yaml` | INFRA | an `infrastructure` endpoint (platform, channel, service, OS, device) | `compromises`, `targets`, `uses` |
| `S5_prose.yaml` | PROSE | fallback: `attack-pattern`, `threat-actor`, `intrusion-set`, `campaign`, `identity`, `location`, `vulnerability` | `attributed-to`, `targets`, `uses` |

Each prompt contains:

- the allowed source type → target type → relationship triples for the group;
- numbered decision rules written with a CTI analyst;
- a few-shot example with fictitious entities;
- the required output schema.

### Confidence scoring (`geval_confidence_score*.yaml`)

These files define the `criteria` and `evaluation_steps` of a custom G-Eval metric named *Correctness*, implemented with the [DeepEval](https://github.com/confident-ai/deepeval) framework. The metric checks whether the predicted relationship type is correct and whether its justification is logically sound and free of unsupported assumptions. It ignores formatting. Each test case pairs the input given to the extraction model with one predicted relationship (identifier, source, target, type and justification). The judge returns a score between 0 and 1 with a short explanation.

## Acknowledgements

Project is co-funded by the European Union under Grant Agreement No. 101190156 and is supported by the European Cybersecurity Competence Centre. The project received funding from the Ministry of Science and Higher Education in Poland.