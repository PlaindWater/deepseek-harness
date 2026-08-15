# Agent Note: Select-all/clear toggle in the Models fetch dialog

Status: implemented

English | [中文](2026-08-15-models-fetch-dialog-select-all-toggle.zh.md)

## Problem

`ModelListEditor` opens the fetch result as a flat picker: every newly discovered candidate starts checked, so a provider that reports many models forces the user to uncheck rows one at a time, and already-configured candidates appear as unchecked rows indistinguishable from the new ones. The [declaration note](../architecture/2026-08-04-declaring-a-provider-from-the-models-page.md) named that second half — "candidates already configured start unchecked" — and it made adopting a selection safe, but the same presentation leaves a re-add entry point through which a picker could still overwrite a capacity the user corrected. The picker needs one bulk control over the newly found group, and the already-configured group needs a presentation that is not a re-add target.

## Decision

`ModelListEditor` splits the discovered candidates into two groups and adds one bulk toggle, all inside the existing client plugin and settings surface; the interrogation mechanism is unchanged (see [interrogating a draft provider endpoint](../architecture/2026-08-04-draft-provider-endpoint-interrogation.md)).

**Two groups with their own rows.** `known` is the set of ids already present in the profile's `models`. `existing` holds discovered ids in `known` and renders each as `ExistingRow` — a disabled, checked checkbox plus the id — under the `fetchGroupExisting` heading; `fresh` holds the rest and renders each as `CandidateRow`, an active checkbox bound to `picked`, under the `fetchGroupNew` heading. `ExistingRow` has no toggle handler and is never part of `picked`, so it is presentational: it shows what is already configured and cannot be re-adopted.

**One toggle flips the newly found group only.** `allPicked` reads `fresh.every(candidate => picked.has(candidate.id))`, is false when `fresh` is empty, and `toggleAll` returns early when `fresh` is empty. Otherwise it writes `picked` minus the `fresh` ids when `allPicked`, and `picked` plus the `fresh` ids when not. The button above the list (`candidateToolbar`, right-aligned) reads `fetchSelectAll` while any `fresh` row is unchecked and `fetchClearAll` once every one is checked.

**The existing re-add guard is strengthened, not replaced.** The earlier decision left configured candidates as active unchecked rows so that adopting never overwrote a corrected capacity; this note makes them non-adoptable outright. That supersedes only the presentation in [declaring a provider from the Models page](../architecture/2026-08-04-declaring-a-provider-from-the-models-page.md), not the guarantee.

## Alternatives considered

**A toggle over the whole candidate list.** This was the first cut. It leaves configured candidates as active unchecked rows, so "select all" re-checks models that are already configured and turns the bulk action into a re-add path.

**Keep configured candidates as active unchecked rows and add only the toggle.** This preserves the prior DOM shape, but the mixed list hides what is already there, and the toggle must then filter them out anyway — splitting into `existing` and `fresh` is the same work with a clearer result.

**Two buttons, "select all" and "clear".** More chrome for a two-state action; one button whose label flips between the two states reads the same way with less surface.

**Change the default so newly found models start unchecked.** Out of scope: the toggle is additive and leaves the fetch's auto-check-new default untouched, because the common case is still "add most of what this endpoint serves."

## Consequences

A provider that reports a long model list is no longer a manual unchecking chore: one click clears the newly found half, and one more restores a full selection. The already-added group is pinned above the new one, so a re-open shows what is configured without mixing it into the new candidates. Adopting a selection cannot overwrite a hand-corrected capacity, now because the configured rows are never adoptable rather than merely starting unchecked.

The costs are local to the picker: two group headings add vertical chrome, and the `ExistingRow` checkbox is disabled rather than interactive, so a capacity the user wants to change is edited on the configured row's own card rather than through the fetch dialog.

## Testing

`packages/client/ui-settings-models/tests/provider-form.client.spec.tsx` gained two cases and edited one: the select-all/clear cycle and adopt in the endpoint-interrogation suite, and already-added-above-newly-found with the scoped toggle leaving the disabled existing row alone across a clear. The stylesheet gate keeps `candidateToolbar` and `candidateGroupHead` referenced from source.