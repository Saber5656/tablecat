# Review resolution addendum

- PR: #1
- Base: `main`
- Resolution scope: the five existing review threads listed below.
- This document is a normative design addendum. It records the accepted resolution contract; it does not claim that implementation or tests have already run.
- Bot review is not retriggered.

## PRRT_kwDOTNkEVs6PFUfM — numeric/boolean first-row header inference

Problem: treating every first row as a header drops valid data when the first row contains numbers or booleans.

Resolution: header inference MUST classify a first row containing numeric or boolean cells, or lacking credible column-name tokens, as data unless an explicit header mode is selected. `--no-header` remains an override. The rule MUST be deterministic and documented.

Focused verification before resolving this thread: parse a numeric/boolean first row such as `1,2\n3,4` and assert both rows remain data; test explicit-header and `--no-header` modes.

## PRRT_kwDOTNkEVs6PFUfP — FWF fallback evidence

Problem: arbitrary prose can be misclassified as fixed-width fields.

Resolution: FWF fallback MUST require at least two aligned columns plus repeated boundary/width evidence across rows. Ordinary prose without stable multi-column alignment MUST return `FormatUnknown` with a remediation message, not a guessed table.

Focused verification before resolving this thread: run aligned multi-row FWF samples and ordinary prose; assert only the former is accepted.

## PRRT_kwDOTNkEVs6PFUfS — finite numeric JSON

Problem: `NaN` and infinities are not valid JSON numeric values.

Resolution: before emitting `TypeNumber`, the parser MUST require a finite value. Non-finite results MUST remain strings or produce a documented validation error, and JSON rendering MUST never emit invalid `NaN`/infinity tokens.

Focused verification before resolving this thread: parse finite, `NaN`, positive infinity, and negative infinity inputs and validate the output with a strict JSON parser.

## PRRT_kwDOTNkEVs6PFUfY — warning propagation through concat

Problem: warnings can be lost when `Parse` output is concatenated.

Resolution: `Parse` and `Concat` MUST return a table plus warning metadata (for example a `ParseResult` wrapper or warnings attached to the table). The CLI MUST emit warnings, including when `--select all`/concat is used; library callers MUST not lose them.

Focused verification before resolving this thread: create an input that produces a warning, concatenate/select all, and assert the warning remains visible in both library and CLI paths.

## PRRT_kwDOTNkEVs6PFUfb — unsupported output renderer flags

Problem: accepting `--to fwf/html` without a renderer creates a false success path.

Resolution: `--to` MUST accept only implemented outputs: `md/markdown`, `csv`, `tsv`, and `json`. `fwf` and `html` MUST be rejected before processing with a nonzero exit and a message listing supported formats. Input formats remain separately governed by `--from`.

Focused verification before resolving this thread: invoke every supported and unsupported output option and assert successful rendering only for the supported set.

## Verification status

The checks above are required acceptance criteria for implementation. This addendum intentionally reports no test result and no implementation-complete status.