# EK Graph — Reconciliation Supplement (from Independent Validation)

## EK-37 over-declaration guard + unparsed fail-safe (soundness self-guard)

- Fact: Phase3 does NOT silently pass when declared authority is too broad to check.
  - self-declared ADMIN on >=3 resources, or a wildcard resource, -> HIGH privilege_escalation finding (A6), message: "least-privilege checking cannot constrain this skill"
  - threshold constant _OVER_DECLARATION_THRESHOLD = 3 (analyzer/engine.py L67)
  - guard block: analyzer/engine.py L376-392
- Fact: unparsable declared capability strings -> LOW capability_violation finding, not silently dropped (analyzer/engine.py L399+)
- Directed execution: ADMIN on 4 resources -> 1 HIGH A6; wildcard -> 1 HIGH; nonsense string -> 1 LOW fail-safe
- epistemic: Fact
- links: [mechanism: EK-04], [constraint: EK-04 complement], [causal: ->C-09]

## EK-38 threat-pattern count and is_safe aggregation semantics

- Fact: patterns.py holds 58 regex patterns (shell/code/prompt-injection/sensitive-env catalogs)
- Fact: AnalysisResult.is_safe aggregates two finding kinds: pattern_match (Phase2 recall-type, e.g. any external URL -> data_exfiltration HIGH) and capability_violation (Phase3 sound-type)
- Directed execution: benign skill (declared network:read + 1 URL) -> safe=False solely due to pattern_match, with NO capability violation; pure filesystem:read benign -> safe=True
- epistemic: Fact (count + execution); semantic implication in C-09
- links: [contrast: EK-05], [causal: ->C-09]

## EK-20 upgrade (Obs -> Fact)

- Independent audit confirmed implementation: claude_skills.py L208-215 extracts allowed-tools/disallowed-tools into capability strings; engine.py L348 resolves them into declared capabilities. Upgraded to Fact.

## EK-05 supplement

- Threat-pattern catalog size: 58 regex patterns (patterns.py count)

## Post-Reconciliation EK stats

- Total EK: 38 (36 + 37 + 38); all 38 have links; 0 isolated; avg out-degree ~2.1
