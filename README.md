# fswatch-test

Scratch repo for validating the Windows-side fswatch -> git two-way sync.
Each fixture below exists to probe one specific failure mode.

| Path | Probes |
| --- | --- |
| `docs/runbook.md` | plain LF text, main edit/collision target |
| `docs/plan/вложенный отчёт.md` | space + Cyrillic in the name |
| `docs/a/b/c/d/deep.md` | deep nesting, recursive watch |
| `data/config.yaml` | structured text edit |
| `data/crlf.txt` | committed with CRLF inside -> autocrlf churn |
| `data/cp1251.txt` | non-UTF8 encoded text |
| `data/blob.zip` | binary, must not be text-converted |
| `data/empty.txt` | zero-byte file |
| `data/big.txt` | ~2 MB, cycle timing |
| `.scratch/` | gitignored, must never be committed |
