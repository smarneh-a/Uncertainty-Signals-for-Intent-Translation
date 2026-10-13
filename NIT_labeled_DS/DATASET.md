# Dataset: Network Intent Translations (NIT) with Ambiguity Labels

## Introduction

This folder contains the data used in our work on uncertainty-aware network intent translation: converting a natural-language network configuration request (an *intent*) into low-level command-line configuration for a Juniper EX3300 switch (JUNOS Base OS 12.3R12-S12) using a fine-tuned Large Language Model (LLM).

The data builds on the **Network Intent Translations (NIT)** dataset and adds **ambiguity labels**, so that model uncertainty can be studied against known levels of intent ambiguity. This README covers:

1. the files in this folder and how the splits were built,
2. the origin of the original dataset and how to cite it,
3. the entry format (fields),
4. how the ambiguity labels are defined, and the label distribution per split.

## Files

| File | Entries | Content |
|---|---|---|
| `nit_train.json` | 847 | Training split. Original NIT entries with ambiguity labels. |
| `nit_val.json` | 94 | Validation split. Original NIT entries with ambiguity labels. |
| `synthesized_test_split.json` | 198 | Test split. **Synthesized** (see below). |



### The test split is synthesized

Train and validation entries come from the original, manually written NIT dataset. The test split is different: its entries were **synthesized** to cover all six ambiguity levels (L0–L5), because the original NIT questions are almost all fully specified and clear (see the distribution table below). Synthesized entries are rewrites of NIT-style intents in which parameter values or task wording are deliberately made vaguer, and they carry their own labels.

- No test question appears in the train or validation split (checked: zero overlap).
- Where an entry was derived from an existing intent, `original_question` and/or `original_answer` keep the source text before synthesis (62 entries have `original_answer`, 5 have `original_question`).

## Original dataset

- **Hugging Face:** <https://huggingface.co/datasets/Smarneh/NIT>
- **Dataset DOI (Digital Object Identifier):** [10.57967/hf/3123](https://doi.org/10.57967/hf/3123)
- **Size:** 1,000 entries, English, JSON format
- **Creation:** written manually from Juniper configuration manuals and network-engineer expertise. Each completion was validated on a physical Juniper EX3300 switch and its syntax checked against the manuals.
- **Command mix:** 582 `set`, 249 `show`, 73 `delete`, and 96 other commands (for example `clear`, `request`, `commit`, `rollback`).

The Hugging Face release has three fields (`question`, `context`, `answer`). All other fields in this folder were added by the research team and are not part of the original release.

## Entry format

| Field | Origin | Description |
|---|---|---|
| `question` | NIT | The network intent in natural language |
| `context` | NIT | Generic form of the required command(s), with placeholders such as `<interface-name>` |
| `answer` | NIT | The configuration that fulfils the intent, using the parameter values given in the question (placeholders remain where the question gives no value) |
| `cmd_category` | added | Command type: `set`, `show`, `delete`, `clear`, or `request` |
| `topic` | added | Configuration topic of the command (for example `system`) |
| `ambiguity_level` | added | Overall ambiguity level L (0–5) |
| `parameter_score` | added | Parameter ambiguity score P (0–2) |
| `description_score` | added | Description ambiguity score R (0–3) |
| `parameter_disambig` | added, optional | Dictionary mapping each missing placeholder to the value that resolves it |
| `parm_answer` | added, optional | The `answer` with the `parameter_disambig` values filled in. Present only for entries with missing parameters. |
| `original_question` | test only, optional | Question text before synthesis |
| `original_answer` | test only, optional | Answer text before synthesis |

Example (test split; some fields shortened):

```json
{
  "question": "I need to compare rollback 3 to the snapshot taken after last week's maintenance.",
  "context": "show system rollback <rollback-number> compare <rollback-number>",
  "answer": "Use the following command in the operational mode:\nshow system rollback 3 compare <rollback-number>",
  "parameter_disambig": {"rollback-number": "1"},
  "parm_answer": "Use the following command in the operational mode:\nshow system rollback 3 compare 1",
  "cmd_category": "show",
  "topic": "system",
  "ambiguity_level": 1,
  "parameter_score": 1,
  "description_score": 0,
  "original_question": "I need to identify what changed in the switch configuration between the backup taken before and after last week's maintenance.",
  "original_answer": "Use the following command in the operational mode:\nshow system rollback <rollback-number> compare <rollback-number>"
}
```

## Ambiguity labeling

The original NIT authors distinguish three prompt types: intents that give all required parameter values, some of them, or none. We extend this idea into a two-part ambiguity rubric. Each intent is scored on two independent dimensions:

- **Parameter score (P, 0–2):** how completely the intent supplies the parameter values the command needs (for example interface name, VLAN identifier, or timer value).
- **Description score (R, 0–3):** how clearly the intent describes the *task itself*, independent of parameters. Higher means the wording is more indirect, so the target command is harder to infer.

The two scores add up to one **ambiguity level**:

> **L = P + R**, giving six levels, **L0** (fully specified, direct) to **L5** (most ambiguous).

This relation holds for every entry in all three files.

<!-- TODO (confirm against the annotation guideline): the level meanings below were inferred from the data. -->

| Score | Level | Meaning |
|---|---|---|
| P | 0 | All required parameter values are given |
| P | 1 | Some required values are missing |
| P | 2 | No required values are given |
| R | 0 | Task stated directly (command-like wording) |
| R | 1 | Mild paraphrase of the task |
| R | 2 | Task described indirectly or through context |
| R | 3 | Only the goal or a symptom is described, with no command-specific wording |

Examples from the test split:

- L3 (P = 2, R = 1): "I want to push my config changes during the maintenance window tonight."
- L5 (P = 2, R = 3): "I need to prevent a specific port from processing topology control frames from neighboring devices."

Keeping P and R separate allows each ambiguity source to be tested on its own, for example parameter-related signals on entries with R = 0, and description-related signals on entries with P = 0.

### Label distribution per split

Number of entries per ambiguity level L:

| Split | L0 | L1 | L2 | L3 | L4 | L5 | Total |
|---|---|---|---|---|---|---|---|
| Train | 702 | 15 | 126 | 4 | 0 | 0 | 847 |
| Validation | 75 | 1 | 18 | 0 | 0 | 0 | 94 |
| Test (synthesized) | 46 | 31 | 31 | 30 | 30 | 30 | 198 |

Number of entries per score:

| Split | P = 0 | P = 1 | P = 2 | R = 0 | R = 1 | R = 2 | R = 3 |
|---|---|---|---|---|---|---|---|
| Train | 719 | 7 | 121 | 827 | 12 | 6 | 2 |
| Validation | 76 | 0 | 18 | 93 | 1 | 0 | 0 |
| Test (synthesized) | 75 | 49 | 74 | 83 | 23 | 43 | 49 |

Train and validation are dominated by L0 because the original NIT questions are mostly clear and complete; the test split is nearly balanced across levels. For analyses that need exactly equal coverage, we use a balanced subset of 192 entries of the test split: 42 at L0 and 30 at each of L1–L5.



## Citation

If you use the NIT dataset, please cite the dataset paper:

```bibtex
@article{samarneh2025nit,
  title   = {{NIT}: A Dataset for Network Intent Translation},
  author  = {Samarneh, Ala' A. and Al Jarrah, Omar Y. and Al-Hammouri, Ahmad T.},
  journal = {Data in Brief},
  pages   = {111842},
  year    = {2025},
  url     = {https://www.sciencedirect.com/science/article/pii/S2352340925005694}
}
```

If you use the ambiguity labels or the synthesized test split, please also cite our work:

```bibtex
@inproceedings{alsamarneh2026uncertainty,
  title     = {Uncertainty Signals for Network Intent Translation: Risk Ranking and Ambiguity Localization},
  author    = {Alsamarneh, Ala' A. and Alhussein, Omar},
  booktitle = {2026 IEEE Globecom Workshops (Workshop on Trust and Intelligence-Native Networks and Systems)},
  year      = {2026}
}
```

## Contact

Questions about the dataset or labels: open an issue in this repository.
