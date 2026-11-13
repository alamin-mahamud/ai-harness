---
plan_sections: Facts Baseline Confidence Steps Rollback Acceptance
---
# Principle: evidence first

Sample house principle. harness-run prepends every `*.md` file in HARNESS_PRINCIPLES_DIR (frontmatter stripped) to every role prompt, and plan-gate-hook requires the headings named in `plan_sections` before it spends a critic call. Replace this text with your own rules. Example: a plan must show raw evidence for each fact (command output, metric, log), a measured baseline for every metric the change should move, a confidence label (HIGH only if the baseline is measured, the problem is proven and a rollback trigger exists), small reversible steps, and an acceptance check that compares before and after. The critic returns REVISE if any of that is missing. A trivial one-step change may say `Baseline: n/a - <reason>`.
