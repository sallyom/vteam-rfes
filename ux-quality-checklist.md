# UX Quality Checklist: LlamaIndex RAG Demo

**Purpose**: Gate checklist for design reviews, draft reviews, and final approval
**Owner**: UX Team Lead (Uma)
**Use**: Check off items as completed, document blockers

---

## Pre-Implementation: Design Review

**Checkpoint**: Before writing code or documentation
**Goal**: Ensure approach aligns with UX standards

### Architecture & Approach
- [ ] Deployment target environment(s) identified
- [ ] User personas and skill levels documented
- [ ] Success metrics defined and measurable
- [ ] Pattern consistency with llama-stack demo verified
- [ ] Technical feasibility confirmed by engineering
- [ ] Resource requirements documented (RAM, disk, time)

### Documentation Plan
- [ ] README structure approved (matches template)
- [ ] Progressive disclosure strategy defined
- [ ] Error scenarios identified and prioritized
- [ ] Sample data strategy decided (provided vs. user-supplied)
- [ ] Visual artifacts needed (diagrams, screenshots) identified

**Reviewer**: ________________  **Date**: __________  **Status**: [ ] Pass [ ] Needs Work

---

## Mid-Implementation: Draft Review

**Checkpoint**: When README and scripts are 50% complete
**Goal**: Catch issues early, provide guidance

### Clarity Standards (UXQ-001 to UXQ-004)
- [ ] Each step has single unambiguous interpretation
- [ ] Prerequisites listed before Quick Start section
- [ ] Every command includes expected output
- [ ] Directory context explicit in every code block
- [ ] No assumed knowledge (all terms defined)

### Completeness Standards (UXQ-005 to UXQ-008)
- [ ] Examples are complete and runnable (not fragments)
- [ ] Happy path documented
- [ ] At least 2 variations/customizations shown
- [ ] Sample data provided or auto-generated
- [ ] Complete workflow demonstrated (ingest → query → results)

### Error Handling Standards (UXQ-009 to UXQ-012)
- [ ] Troubleshooting section exists
- [ ] At least 5 common errors documented
- [ ] Each error has remediation steps
- [ ] Setup scripts validate prerequisites
- [ ] Error messages are actionable (not just descriptive)
- [ ] Health check command provided

### Consistency Standards (UXQ-013 to UXQ-016)
- [ ] Directory structure matches llama-stack pattern
- [ ] README follows template structure
- [ ] Code naming follows conventions
- [ ] Documentation tone consistent with other demos

### User Journey Quality (UXJ-001 to UXJ-016)
- [ ] Quick Start section appears first
- [ ] Maximum 5 commands in happy path
- [ ] Progressive disclosure: Quick → Details → Advanced
- [ ] Setup script is idempotent (safe to run twice)
- [ ] Cleanup commands documented
- [ ] Progress indicators for long operations (>30 sec)
- [ ] Success confirmations show next steps

### Documentation UX (DOCUX-001 to DOCUX-015)
- [ ] README structure matches mandatory order
- [ ] Heading hierarchy correct (H1 once, H2 sections, H3 subsections)
- [ ] Commands use \`code style\`
- [ ] Imperative mood for instructions
- [ ] Second person ("you") used throughout
- [ ] Action-first language (lead with verbs)
- [ ] Outcomes specified for each step

**Issues Identified**:
1. _______________________________________________
2. _______________________________________________
3. _______________________________________________

**Reviewer**: ________________  **Date**: __________  **Status**: [ ] Pass [ ] Needs Work

---

## Pre-Merge: Usability Test

**Checkpoint**: When feature is 90% complete
**Goal**: Fresh-eyes validation on real environment

### Test Environment
- [ ] Clean RHEL 9 VM prepared (no prior setup)
- [ ] Test user has no prior exposure to demo
- [ ] Screen recording started (for analysis)
- [ ] Test user has access to demo repository only

### Test Protocol
1. Give user only the README
2. Ask them to follow Quick Start
3. Do NOT provide help (observe and note issues)
4. Time to first success
5. Post-test survey

### Observations
- [ ] User completed Quick Start without asking questions
- [ ] All commands executed successfully on first attempt
- [ ] User understood what they built
- [ ] Time to first success: _______ minutes (target: ≤10)
- [ ] Number of errors encountered: _______ (target: 0)

### Error Log
| Step | Error Encountered | User Reaction | Root Cause |
|------|------------------|---------------|------------|
|      |                  |               |            |
|      |                  |               |            |
|      |                  |               |            |

### User Confidence Survey (1-5 scale, 5=strongly agree)
- [ ] "I understand what LlamaIndex RAG does": _____
- [ ] "I could modify this for my own documents": _____
- [ ] "I know how to troubleshoot if something breaks": _____
- [ ] "I could explain this demo to a colleague": _____
- [ ] "The documentation was clear and easy to follow": _____
- [ ] "I felt confident at each step": _____

**Average Score**: _____ (target: ≥4.0)

**Critical Issues** (block merge):
1. _______________________________________________
2. _______________________________________________

**Minor Issues** (fix before merge):
1. _______________________________________________
2. _______________________________________________

**Tester**: ________________  **Date**: __________  **Status**: [ ] Pass [ ] Needs Work

---

## Pre-Merge: Final UX Review

**Checkpoint**: Before merging to main branch
**Goal**: Comprehensive quality verification

### Success Metrics (METRIC-001 to METRIC-012)
- [ ] Time to first success: ≤10 minutes (verified in usability test)
- [ ] Number of steps in happy path: ≤5 commands
- [ ] User confidence survey average: ≥4.0
- [ ] All common failure points documented with solutions
- [ ] Prerequisites have automated verification
- [ ] Setup is idempotent and cleanup is available

### Customer Considerations (CUST-001 to CUST-020)
- [ ] Environment requirements documented (local, cloud, airgapped)
- [ ] Sample documents include variety of types
- [ ] Document size guidance provided (small, medium, large)
- [ ] Skill level paths defined (beginner, intermediate, expert)
- [ ] Glossary of terms provided
- [ ] Architecture diagram included
- [ ] Accessibility: Plain language (Flesch Reading Ease >60)
- [ ] Accessibility: Alt text for all diagrams
- [ ] Accessibility: Inclusive language (no ableist terms)
- [ ] Accessibility: Screen reader friendly formatting

### Documentation Completeness
- [ ] README exists and follows template
- [ ] Quick Start section complete
- [ ] Prerequisites section complete with verification
- [ ] Usage examples section complete (≥3 examples)
- [ ] Architecture section complete with diagram
- [ ] Troubleshooting section complete (≥5 scenarios)
- [ ] Advanced topics section present
- [ ] Sample data attribution documented
- [ ] Contributing guidelines linked

### Code Quality (Scripts & Examples)
- [ ] All scripts have descriptive docstrings
- [ ] All scripts validate prerequisites before execution
- [ ] All scripts provide progress feedback
- [ ] All scripts have clear error messages
- [ ] All examples are complete and runnable
- [ ] All examples include expected output
- [ ] Code follows PEP 8 style guidelines
- [ ] No hardcoded credentials or secrets

### Pattern Consistency
- [ ] Directory structure matches llama-stack demo
- [ ] File naming matches established conventions
- [ ] README structure matches template
- [ ] Code style matches existing demos
- [ ] Documentation tone consistent across demos

### Accessibility Compliance
- [ ] Plain language verified (automated readability check)
- [ ] All diagrams have alt text
- [ ] No color as sole indicator of meaning
- [ ] Terminal output uses symbols, not just colors
- [ ] Inclusive language verified (no problematic terms)
- [ ] Commands work with screen readers

### Testing Verification
- [ ] Fresh machine test passed (clean VM)
- [ ] All Quick Start commands executed successfully
- [ ] All usage examples tested and work
- [ ] All troubleshooting scenarios verified
- [ ] Error messages tested and accurate
- [ ] Cleanup script tested and works

### Review Sign-offs
- [ ] Technical writer reviewed and approved
- [ ] UX team lead reviewed and approved
- [ ] Accessibility review completed
- [ ] Security review completed (no secrets, safe scripts)
- [ ] Product owner approved
- [ ] Engineering lead approved

**Blocking Issues**:
1. _______________________________________________
2. _______________________________________________

**Reviewer**: ________________  **Date**: __________  **Status**: [ ] APPROVED [ ] BLOCKED

---

## Post-Merge: Continuous Improvement

**Goal**: Track actual user experience, iterate based on feedback

### Metrics Collection (30 days post-launch)
- [ ] Time to first success tracked (if telemetry enabled)
- [ ] GitHub issues/questions reviewed
- [ ] Common support requests categorized
- [ ] User feedback collected (surveys, comments)
- [ ] Demo usage analytics reviewed (if available)

### Improvement Areas Identified
| Issue | Frequency | Severity | Proposed Fix |
|-------|-----------|----------|--------------|
|       |           |          |              |
|       |           |          |              |
|       |           |          |              |

### Action Items
- [ ] Update documentation based on user feedback
- [ ] Add troubleshooting entries for new issues
- [ ] Enhance examples based on common use cases
- [ ] Improve error messages for common failures
- [ ] Update prerequisites if environment changed

**Next Review Date**: __________

---

## Scoring Rubric

Use this for quantitative assessment:

| Category | Weight | Score (1-5) | Weighted |
|----------|--------|-------------|----------|
| Clarity | 25% | _____ | _____ |
| Completeness | 25% | _____ | _____ |
| Error Handling | 20% | _____ | _____ |
| Consistency | 15% | _____ | _____ |
| Accessibility | 15% | _____ | _____ |
| **TOTAL** | **100%** | | **_____** |

**Minimum passing score**: 4.0

### Score Definitions:
- **5 - Exemplary**: Publish as template for future demos
- **4 - Complete**: Meets all requirements, ready to merge
- **3 - Acceptable**: Minor gaps, document improvements before merge
- **2 - Needs Work**: Major gaps, requires revision and re-review
- **1 - Incomplete**: Not ready for review, needs significant work

---

## Checklist History

| Date | Checkpoint | Reviewer | Status | Notes |
|------|------------|----------|--------|-------|
|      | Design Review |        |        |       |
|      | Draft Review |        |        |       |
|      | Usability Test |        |        |       |
|      | Final Review |        |        |       |
|      | Post-Merge Review |        |        |       |

---

## Notes & Comments

**Design Review Notes**:


**Draft Review Notes**:


**Usability Test Notes**:


**Final Review Notes**:


**Post-Merge Notes**:


---

**Document Version**: 1.0
**Last Updated**: 2025-10-21
**Owner**: Uma, UX Team Lead
