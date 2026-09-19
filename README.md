# matura.lol datasets

Open datasets behind **[matura.lol](https://matura.lol)** — a read-only
full-text search engine over the Polish exam corpus (CKE matura, egzamin
ósmoklasisty, egzamin gimnazjalny, vocational arkusze and próbne arkusze).

Source PDFs (CKE/OKE arkusze) are **not** included, and neither are any images.
Everything here is JSON, produced by the pipeline; large files are zstd
(`.zst`) — `zstd -d <file>.zst` or `unzstd <file>.zst`.

## Layout

```
imports/    processed enrichments folded onto the corpus
            inventory.json          paper/PDF metadata + hashes
            math_import.json        curated math index
            site_import.json        site problems + SVG solutions
            odrabiamy_import.json   answers/worked solutions (odrabiamy)
            matematykaorg_import.json
            maturazai_import.json   AI answers/solutions
            zadaniazmatur_import.json
            maturaonline_import.json
            szybkiekorepetycje_import.json
            biologhelp_import.json  CKE marking schemes (bio/chem)
            pytania_import.json
sources/    raw scraped ledgers (one JSONL row per task/arkusz)
corpus/     segments.tar.zst — per-paper question segmentation
            (question text, page + y span, points, type, key links)
jev/        TypeSafe Jev tagging outputs (see below)
```

## How to navigate this repo

* **Format.** Everything is JSON — either plain `.json`/`.jsonl` or zstd
  compressed (`.zst`). To read a `.zst` file:
  `zstd -d tags.jsonl.zst` (or `unzstd tags.jsonl.zst`). `jq` works on the
  plain output; `pandas.read_json(path, lines=True)` reads the JSONL rows.
* **The corpus.** The per-paper question segmentation is bundled as one archive:
  `tar -I zstd -xf corpus/segments.tar.zst` produces `data/segments/*.json`,
  one file per paper (the paper id from `corpus/inventory.json`).
* **IDs are the join key.** Every question has an id of the form
  `<paper-id>/zad/<number>` — the same id appears in `segments/`, in
  `imports/*.json`, and in `jev/tags.jsonl` (`tags.jsonl.zst`). The
  `imports/*.json` files map per-source enrichments (answers, solutions,
  topics) onto those ids; `jev/*.jsonl` maps the Jev decisions onto them.
* **Where each answer/solution comes from.** A question row carries
  `answer_source` / `answer_text_source` / `solution_source` / `topics_source`
  (values like `cke`, `odrabiamy`, `matematykaorg`, `maturazai`, `ai`,
  `jev`) — see `imports/` per source.
* **The Jev layer** is optional enrichment: join `jev/tags.jsonl.zst` by `id`
  to get topic/difficulty/method/group, `jev/answer_checks.jsonl` for the
  answer-fit signal, and `jev/mismatches.jsonl` + `jev/reassignments.jsonl`
  for the flagged/repair rows.

## jev/

Jev (System One) tags each question with typed decisions: the official
**dział** (from the CKE informatory), **difficulty**, **solution method**,
**answer form**, **bloom** level, estimated **time**, and capability/tool
flags. See https://github.com/matura-lol/Jev-categorise for the tooling.

| file | rows | what |
| --- | --- | --- |
| `informator_topics.json` | — | official per-subject dział taxonomy extracted from the CKE informatories |
| `tags.jsonl` | ~47k | one Jev tag record per question (topic + probabilities, difficulty, method, answer form, bloom, time, capabilities, capability vector, group_id) |
| `tags.sample.jsonl` | 500 | plain-text sample of the same schema |
| `answer_checks.jsonl` | ~41k | per-question Noul: does the stored answer/solution fit the task? |
| `mismatches.jsonl` | — | the low-probability subset of `answer_checks` for review |
| `reassignments.jsonl` | — | proposals matching a mismatched answer to the question it actually solves (one-to-one, never applied automatically) |

### Tag schema (one row)

```json
{
  "id": "informator-maturalny-matematyka-2023-poziom-podstawowy/zad/19.2",
  "subject": "matematyka",
  "topic": "Funkcje",
  "topic_confidence": 1.0,
  "topic_probs": {"Funkcje": 1.0, "...": 0.0},
  "difficulty": 2,
  "difficulty_label": "średnie",
  "method": "rachunek",
  "answer_form": "liczba",
  "capabilities": {"wymaga_rachunku": 0.98, "...": 0.0},
  "bloom": 3,
  "time_band": 0,
  "needs_formula_sheet": 0.2,
  "needs_calculator": 0.1,
  "visual": 0.08,
  "subject_ok": 0.99,
  "vector": [0.0, 1.0, "..."],
  "group_id": "matematyka/funkcje/rachunek/2/1e",
  "group_label": "Funkcje · rachunek · średnie",
  "model": "jev-1.13",
  "input_tokens": 2612
}
```

`group_id` is the discrete signature `subject/topic/method/difficulty/capability-bits`;
questions sharing it are solved the same way.

## Source & licence

The dataset (the compilation, the segmentation, and the model-derived metadata)
is released under the **GNU Affero General Public License v3.0** — see
[`LICENSE`](LICENSE).

Individual pieces of **text** inside the data may be licensed differently and
remain under their respective licences; the AGPL does not override them.
In particular:

- exam papers and marking schemes are public materials of the **Centralna
  Komisja Egzaminacyjna (CKE)** and the **Okręgowe Komisje Egzaminacyjne (OKE)**
  and remain theirs;
- question text, answers and worked solutions collected from third-party sites
  (odrabiamy, matematyka.org.pl, zadaniazmatur, matura-online, biologhelp,
  szybkiekorepetycje, maturazai and others) remain subject to those sites'
  terms and licences;
- the code that produced this data lives at
  https://github.com/matura-lol/Jev-categorise (MIT).

If you reuse this data, credit [matura.lol](https://matura.lol) with a link.
