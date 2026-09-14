# Escaped bugs — the postmortem loop

The real scorecard of a QA process is not the bugs it found; it's the ones that
got past it.

Read this when the user reports a bug discovered in production — or anywhere
downstream of a run that should have caught it. Run the loop alongside fixing the
bug, if fixing was asked for, never instead of it.

1. **Attribute it.** Which past run owned the surface this bug lives in? Which
   catalogue category and tier would have caught it?
2. **Diagnose the miss** — exactly one of these, named in writing:
   - *Design gap*: no case pointed at it → the technique or catalogue walk
     failed. Why did the matrix not generate it?
   - *False PASS*: a case covered it and passed → the assertion was hollow, the
     fixture was wrong, or a subagent's word was taken untested.
   - *Known but parked*: it was `BLOCKED`/`NOT RUN` and never re-queued → the
     debt-tracking failed.
   - *Out of scope*: the level or tier plan excluded the area → was that
     exclusion reasonable with what was known then? (Sometimes yes — say so.)
3. **Close the hole.** Write the regression case with a new ID into the suite,
   add the bug to `.qa/known-issues.md`, and increment the `escaped` column of
   the run that missed it in `.qa/metrics.md`.
4. **Generalise.** If the miss pattern would recur in other projects, it's a
   prima facie **P1** proposal in `RETROSPECTIVES.md` — an escaped bug teaches
   more than ten found ones, precisely because it beat the whole process.
