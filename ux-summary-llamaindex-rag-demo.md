# UX Summary: LlamaIndex + pgvectordb RAG Demo

**Quick Reference for RFE Development**
**Prepared by**: Uma, UX Team Lead
**Date**: 2025-10-21

---

## TL;DR: Critical UX Requirements

This demo is an **educational product** where the documentation IS the feature. Quality gates are strict.

### Must-Haves:
1. **10-minute success path** - Clone to working demo in ≤10 minutes
2. **Zero-ambiguity instructions** - Every command has one interpretation
3. **Proactive error prevention** - Scripts validate prerequisites before execution
4. **Pattern consistency** - Matches llama-stack demo structure exactly
5. **Accessibility compliance** - Plain language, inclusive design

---

## Key UX Standards (Numbers Reference Full Doc)

### Clarity (UXQ-001 to UXQ-004)
- Single unambiguous interpretation per step
- Prerequisites explicitly stated with verification commands
- Expected outputs shown for every command
- Directory context explicit in every code block

### Completeness (UXQ-005 to UXQ-008)
- Complete runnable examples (not fragments)
- Happy path + common variations documented
- Sample data provided (no "bring your own")
- Full value chain demonstrated (ingest → query → results)

### Error Handling (UXQ-009 to UXQ-012)
- Troubleshooting section with all common errors
- Prerequisites validated before execution
- Error messages include context + remediation
- Health check command provided

### Consistency (UXQ-013 to UXQ-016)
- Directory structure matches llama-stack pattern
- README structure matches established format
- Code style and naming conventions aligned
- Documentation tone consistent across demos

---

## User Journey Design (UXJ-001 to UXJ-016)

### Happy Path Priority:
```markdown
# In README - Quick Start comes FIRST
1. Prerequisites verification
2. One-command setup
3. Start services
4. Run demo
5. See results
```

### Progressive Disclosure:
- **5 min**: Quick Start (working demo)
- **15 min**: Customization (your documents)
- **30 min**: Deep dive (architecture)
- **Reference**: Production considerations

### Error Prevention:
- Idempotent setup (safe to run twice)
- Guard rails on destructive operations
- Path validation before execution
- Clear cleanup commands

### Feedback at Every Step:
- Progress indicators for long operations
- Success confirmations with next steps
- Real-time logs during execution
- Status check commands available

---

## Documentation UX (DOCUX-001 to DOCUX-015)

### README Structure (Mandatory Order):
1. One-line description
2. Quick Start
3. What You'll Build
4. Prerequisites (detailed)
5. Usage Examples
6. Architecture
7. Troubleshooting
8. Advanced Topics

### Writing Style:
- **Imperative mood**: "Run the script" (not "The script can be run")
- **Second person**: "You can customize..." (not "Users may...")
- **Action-first**: Lead with the verb
- **Outcome-specified**: Tell users what they'll achieve

### Scanability:
- Front-load key info in each section
- Use parallel structure for options
- Strategic whitespace
- Search-optimized headers (use terms people search for)

---

## Success Metrics (METRIC-001 to METRIC-012)

### Time to First Success
- **Target**: 10 minutes (90th percentile)
- **Stretch**: 5 minutes (50th percentile)
- **Measure**: Clone → successful query

### Number of Steps
- **Target**: ≤5 commands in happy path
- **Optimization**: Automate all automatable steps

### User Confidence (Survey >80% agree)
- "I understand what LlamaIndex RAG does"
- "I could modify this for my own documents"
- "I know how to troubleshoot issues"
- "I could explain this to a colleague"

### Zero Preventable Failures
1. Missing prerequisites → Automated verification
2. Wrong directory → Scripts check location
3. Port conflicts → Scripts check availability
4. Stale state → Cleanup utility provided
5. Version mismatches → Dependencies pinned

---

## Customer Considerations (CUST-001 to CUST-020)

### Environment Support:
| Environment | Key Consideration |
|------------|-------------------|
| Local Dev | Quick setup, resource constraints |
| OpenShift | Network policies, persistent storage |
| Cloud VM | Firewall rules, SELinux |
| Air-gapped | Offline docs, vendored dependencies |

### Skill Levels:
- **Beginner**: Glossary, diagrams, "what just happened" explanations
- **Intermediate**: Configuration reference, tuning guide
- **Expert**: Security, scalability, production checklist

### Accessibility:
- Plain language (Flesch Reading Ease >60)
- Alt text for diagrams
- Screen reader friendly
- Inclusive language (avoid ableist terms)
- International considerations (simple English)

---

## Quality Gates (Before Merge)

Must pass ALL of these:

- [ ] **Fresh machine test** - Clean RHEL 9 VM, no prior setup
- [ ] **10-minute target** - New developer completes Quick Start
- [ ] **Documentation review** - Technical writer approval
- [ ] **UX review** - Clarity and completeness check
- [ ] **Accessibility review** - Inclusive design verification
- [ ] **Consistency review** - Matches llama-stack pattern
- [ ] **Error scenario coverage** - All common failures documented
- [ ] **Confidence survey** - >80% on all confidence questions

---

## Design Review Checkpoints

Tag @uma for review at these stages:

1. **Design Review** - Architecture and approach (before implementation)
2. **Draft Review** - Initial README and scripts (50% complete)
3. **Usability Test** - Fresh-eyes testing (90% complete)
4. **Final Review** - Complete documentation (before merge)

---

## Critical Success Factors

These determine whether this demo succeeds:

1. **Documentation quality** - Since docs ARE the product
2. **Error prevention** - Proactive rather than reactive
3. **Pattern consistency** - Leverages existing mental models
4. **Progressive disclosure** - Fast start, deep learning available
5. **Inclusive design** - Accessible to all skill levels and abilities

---

## Open Questions for Team Discussion

Answer these during RFE development:

1. **Deployment priority**: Local, OpenShift, cloud, or all?
2. **Document corpus**: Provide samples or use existing?
3. **Vector database**: PostgreSQL+pgvector or separate service?
4. **Model selection**: Which LLMs to support?
5. **Performance targets**: Acceptable query latency?
6. **Integration scope**: LlamaIndex only or show tool integrations?

---

## References

- **Full UX Input**: `/workspace/sessions/agentic-session-1761065049/workspace/vteam-rfes/ux-input-llamaindex-rag-demo.md`
- **Existing Pattern**: `/workspace/sessions/agentic-session-1761065049/workspace/docs2db-api/demos/llama-stack/README.md`
- **Spec Template**: `/workspace/sessions/agentic-session-1761065049/workspace/vteam-rfes/.specify/templates/spec-template.md`

---

## UX Team Availability

- **Design critique sessions**: Tuesdays and Thursdays
- **Documentation reviews**: 2-day turnaround
- **Usability testing**: Requires 1-week notice
- **Contact**: Tag @uma in PR or Slack

---

**Remember**: This is a demo, but users judge product quality by demo quality. Make it exceptional.
