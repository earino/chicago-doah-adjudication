# Verification

What was checked, how, and what it does **not** establish.

Scope: this release was built, gated, baselined and package-checked by the publisher, and is
verified below from a fresh consumer environment. Independent verification by someone else has
**not** happened, and nothing here should be read as if it had.

## 1. The artifact can be rebuilt

`code/build.py` was run a second time, on a different machine from the one that produced the
release, against the same live source and with the same pinned `--built-at`:

| file | bytes | sha256 |
|---|---|---|
| `public/train.csv` | 3,751,973 | `67b2a58379b4d1ca8eb11670f9f8a51b1fe15fb4945a173a805a5f474cd154cb` |
| `public/eval.csv` | 267,311 | `bcecdbff8018f74c01697229591b9e9c87b5206c8d61d8f7ef77fa37669c967e` |
| `private/holdout.csv` | 226,237 | `e9855daa7a7d523ea00ad2184867d73d1bf747d4c0d718a8398da5acf749e8be` |
| `meta.json` | 2,300 | `696730446f8bb10c9d7e26a6b1eb8dae46843e887d62846c38081469a9743979` |
| `quality.json` | 3,803 | `7ee12a3db4dbd00bf336f3f96b220b260f6e9b463f0cd7fc41a5a58cb527d882` |

All five digests match the release files **and** match the digests the worker reported at build
time (`source_rows_sha256 9ac0c2ab…`, `case_table_sha256 e0278fe1…`), so the rebuild reproduced the
source-row stream and the reduced case table, not merely the output bytes.

`artifact_version = b71586bf28fa56ebbfaede26c4494d68b166545ac504b2ebd8a52c7f272627ef`
(BLAKE2b over the sorted `name → sha256` mapping of the five files.)

The same builder was run twice on two separate workers (`chicago-001`, `chicago-002`) and produced
identical source and case-table digests, so the build is not sensitive to request timing or paging
order.

## 2. The gate passed

`skills/dataset-qualification/scripts/qualify_dataset.py`, version 1.3.0, run **on the worker
against the full 53,763-case artifact** (not a sample): **32 checks, 0 failed**. The full JSON
verdict is `qualification.json`, reproduced from the job's own output.

One caveat worth stating plainly, because it nearly shipped: **two earlier artifacts of this same
dataset passed this same gate while leaking.** They shipped `hearing_date` and a derived
`days_to_hearing` feature declared as ordinary features, and `days_to_hearing` separates the classes
on its own (single-column AUC 0.84 over the case rows). The gate's post-hoc rule keys off the
artifact's own declarations, so a wrong declaration passes. The leak was caught by reading the built
CSV header, not by the gate — see `RELEASE_NOTES.md` and `notes.md`. A passing gate is evidence
about declarations, not about columns.

## 3. The benchmark runner was not modified

One baseline was run through the runner's **existing** contract, with the three runner files
verified by hash before use:

| file | sha256 | status |
|---|---|---|
| `train.py` | `a3c6bcf13735bc85c52129ded68f839090dffdc266ebc3810dc61b2e6ea5e7e8` | unmodified |
| `validate.py` | `b597e7f84fed614e64b4a86fbecbd6ec0145916a24d0d221a796586976418e7b` | unmodified |
| `validate.sh` | `3f06ca48f11c2e05d331b4d8394284925de660c1d4b086d2f8243959e83b7e6d` | unmodified |

Run on a worker that rebuilt the artifact first and **refused to measure it unless the rebuild was
byte-identical** to `expected_artifact.json`.

## 4. Baseline

`chicago-baseline-001`, both paths agreeing:

- `train.py` → `Eval AUC: 0.6915`
- `validate.sh` → `validate.py` re-predicted from the persisted model, `Eval AUC: 0.6915`
- `contract_ok: true`; Python 3.13.15, pandas 2.3.3, numpy 2.5.3, xgboost 3.4.1, scikit-learn 1.9.1

**Read that against eval's own 0.8955 positive rate, not against 0.5.** A constant-`Liable`
predictor already scores 89.6% accuracy on `eval`; an AUC of 0.6915 is a genuine floor above
chance, but this is a hard, drifting-population task and the number should not be dressed up. No
agent or harness comparison was run, and none is implied.
