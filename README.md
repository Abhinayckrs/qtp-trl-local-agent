# qtp-trl-local-agent
# Local LLM Agent for TRL-Driven QTP Updates

> **Status: Proof of Concept / Experimental**

A local/offline LLM-based agent architecture for assisting with updates to aerospace Qualification Test Procedure (QTP) documents based on a current Test Requirements List (TRL).

The project investigates whether a relatively small local language model can:

* understand engineering test requirements,
* semantically locate the relevant QTP content,
* identify changes even when Test IDs do not directly appear in the QTP,
* handle newly introduced requirements,
* propose surgical document edits,
* challenge its own proposed mappings,
* and eventually generate an updated QTP while preserving unrelated content.

**This project is experimental. It is not a certified aerospace engineering tool and should not be used for unsupervised release of engineering documentation.**

---

## Why This Project?

The target workflow is conceptually:

```text
Current TRL
    +
Existing QTP
    ↓
Understand requirement
    ↓
Find relevant QTP content
    ↓
Determine:
    - update existing
    - no change
    - add requirement
    - new section
    - needs human review
    ↓
Generate surgical document edits
    ↓
Human approval / controlled application
```

A key requirement is that the LLM should perform the semantic reasoning.

The system should **not** depend on a hard-coded mapping such as:

```text
T06 → QTP Section 4.10
```

because new requirements may not have the same Test ID or wording as the existing QTP.

---

# Current Project Constraints

The real target workflow has the following characteristics:

* QTP documents can be approximately 50 pages.
* TRLs can contain up to approximately 20 requirements.
* The first several rows of the TRL may be metadata.
* The actual TRL header may begin around row 6.
* A `Description` field exists.
* Engineering descriptions may contain tolerances.
* TRL-driven changes occur in Chapter 4 at maximum.
* Completely new TRL requirements can appear.
* New requirements may need to be mapped to an existing QTP section.
* The system must not assume that the TRL Test ID exists in the QTP.

Real company documents are confidential and are not included in this repository.

---

# Current Model

Initial experiments used:

**Qwen/Qwen3-4B-Instruct-2507**

The model was successfully run locally in a Colab GPU environment using 4-bit quantization.

The first CPU attempt exhausted available memory.

---

# Experimental Architecture

The current promising architecture is multi-agent:

```text
                 TRL Requirement
                       |
                       v
              +------------------+
              | Engineering      |
              | Analyst LLM      |
              +--------+---------+
                       |
                       v
              +------------------+
              | QTP Investigator |
              | LLM              |
              +--------+---------+
                       |
                       v
              +------------------+
              | Edit Planner LLM |
              +--------+---------+
                       |
                       v
              +------------------+
              | Contrarian       |
              | Reviewer LLM     |
              +--------+---------+
                       |
                       v
              +------------------+
              | Final Judge LLM  |
              +--------+---------+
                       |
              +--------+--------+
              |                 |
             PASS          NEEDS_REVIEW
              |                 |
              v                 v
         Apply Edit          Human
              |
              v
        Updated QTP
```

The same local model can be used for multiple agent roles.

The roles are intentionally separated because a single LLM call can make a plausible but incorrect semantic mapping.

---

# Why a Contrarian Reviewer?

One of the most important findings from the experiments was that a 4B model can be confidently fooled by misleading document content.

In an early stress test, the model incorrectly mapped a random vibration requirement to a thermal-shock section because misleading random-vibration-like table content appeared in the wrong location.

The model selected the wrong section with high confidence.

This demonstrated that:

> numerical/keyword similarity can sometimes overpower engineering context in a small model.

The multi-agent design therefore includes an independent reviewer whose job is explicitly to try to prove the proposed mapping wrong.

---

# Synthetic Benchmark

The project uses synthetic data for experimentation.

Initial benchmark files:

```text
TRL_OLD.xlsx
TRL_NEW.xlsx
OLD_QTP.docx
GROUND_TRUTH.json
AGENT_PROMPT.txt
README.md
```

The synthetic TRL has metadata rows followed by:

```text
Test ID
Description
Requirement
Tolerance
Test Method
Configuration
```

The benchmark includes changes to operating temperature, thermal shock, vibration, output voltage and endurance requirements.

It also includes synthetic new requirements for connector retention, mounting-interface pull strength and connector electrical continuity.

---

# Initial TRL Difference Test

The initial benchmark contained seven deliberate changes between the old and new TRL.

The seven changes were:

1. T01 temperature range
2. T01 temperature tolerance
3. T03 thermal-shock temperature range
4. T03 transition rate
5. T06 vibration level
6. T10 output-voltage tolerance
7. T13 endurance duration

The mechanical TRL comparison correctly detected all seven changes.

This demonstrated that the benchmark plumbing could correctly identify known changes.

---

# Single-Agent 4B Results

A single Qwen3-4B model was first tested against the QTP.

It successfully mapped the known changes at a high level.

However, when given a larger Chapter 4 stress-test document containing misleading/competing content, it could be confidently wrong.

### Important failure

For the random vibration requirement:

```text
T06
10–500 Hz
7.0 grms
```

the model selected:

```text
4.9 Thermal Shock / Rapid Temperature Transition
```

instead of:

```text
4.10 Random Vibration Verification
```

The reason was that misleading random-vibration-like content appeared under the wrong synthetic section.

This is an important limitation of the single-agent approach.

---

# Multi-Agent 4B Results

The same general task was then tested using:

1. Engineering Analyst
2. QTP Investigator
3. Contrarian Reviewer
4. Final Judge

The multi-agent architecture successfully resolved the previous T06 failure.

### Results

| Test | Requirement                                    | Result         |
| ---- | ---------------------------------------------- | -------------- |
| T01  | Operating temperature                          | 4.4 — correct  |
| T03  | Thermal shock                                  | 4.9 — correct  |
| T06  | Random vibration                               | 4.10 — correct |
| T10  | Output voltage                                 | NEEDS_REVIEW   |
| T13  | Functional endurance                           | 4.20 — correct |
| T16  | Connector retention                            | 4.13 — correct |
| T17  | Mounting pull strength                         | 4.12 — correct |
| T18  | Connector continuity after mechanical exposure | 4.13 — correct |

The T06 result is particularly significant because the same requirement had previously been incorrectly mapped by the single-agent system.

---

# Why T10 Was Considered a Good Failure

T10 required:

```text
5.0 VDC ±0.05 V
```

The QTP section was only vague about a "specified voltage range."

The final judge returned:

```text
NEEDS_REVIEW
```

instead of pretending that the QTP contained the exact requirement.

This is desirable behavior for an engineering-document system.

A useful engineering agent should be able to say:

> "I cannot establish this from the QTP evidence."

rather than hallucinating a compliant interpretation.

---

# Exact Edit Experiment

After semantic mapping, the system was tested on actual edit planning.

Three cases were selected:

* T01 — existing requirement with changed values
* T03 — existing requirement with multiple changed values
* T16 — completely new requirement

---

## T01 Edit

The system identified:

```text
4.4 Operating Temperature Verification
```

and identified the exact existing paragraph.

It proposed incorporating:

```text
-30°C to +70°C
±2°C
```

The independent reviewer approved the edit.

The final agent produced a surgical `REPLACE` operation.

---

## T03 Edit

The system identified:

```text
4.9 Thermal Shock / Rapid Temperature Transition
```

and correctly captured both:

```text
-30°C → +70°C
```

and:

```text
3–6°C/min
```

The reviewer approved the edit.

The final agent produced a surgical `REPLACE` operation.

This is important because the requirement contained multiple independent changes.

---

## T16 New Requirement

T16 introduced:

```text
Connector retention
50 N axial retention force
±5 N
Mechanical pull test
```

The initial agent considered it a new section but identified:

```text
4.13 Connector and Interface Integrity
```

as the appropriate location.

The reviewer agreed.

The final judge correctly changed the operation to an insertion into the existing section rather than creating a completely new section.

This demonstrated that:

> New TRL requirement ≠ necessarily new QTP section.

---

# What Has Been Demonstrated

The current synthetic experiments demonstrate that a local 4B model can:

* interpret engineering-style test requirements,
* semantically map requirements to QTP sections,
* work without relying on matching Test IDs,
* handle changed numerical requirements,
* handle multiple changes in one requirement,
* identify existing QTP homes for new requirements,
* detect some cases where QTP evidence is insufficient,
* use a separate reviewer to challenge mappings,
* identify paragraph-level edit targets,
* generate surgical replacement/insertion instructions.

---

# What Has NOT Been Demonstrated

This project has NOT yet proven:

* production reliability,
* reliability on real company QTPs,
* reliable performance on arbitrary 50-page QTPs,
* reliable table editing,
* preservation of complex Word formatting,
* preservation of headers/footers/cross-references,
* tracked-changes compatibility,
* scanned-document/OCR handling,
* robustness against all contradictory engineering content,
* aerospace certification or compliance,
* suitability for unsupervised engineering-document release,
* that 4B is the optimal model,
* that multi-agent architecture always improves results,
* that the model will never hallucinate,
* that all real TRL-driven changes can always be mapped automatically.

---

# Known Synthetic-Benchmark Limitation

The first 10-page stress document contained some synthetic table/section alignment problems.

Therefore, some adversarial behavior in that document was partly caused by the benchmark itself.

Future benchmarking should use:

1. Clean document
2. Deliberately adversarial document
3. New-requirement document
4. Ambiguous document

This will separate model limitations from benchmark construction problems.

---

# Most Important Finding

The most important finding so far is:

> A small local 4B model that can be confidently wrong in a single-agent document-location task became substantially more useful when the workflow was changed to include independent analysis, investigation, adversarial review and final judgment.

This does NOT prove production readiness.

It does justify continuing the architecture experiment.

---

# Current Architecture Recommendation

The current recommended direction is:

```text
Current TRL
    ↓
LLM requirement analysis
    ↓
LLM semantic QTP investigation
    ↓
LLM exact edit planning
    ↓
Independent LLM review
    ↓
Final LLM decision
    ↓
Human approval where necessary
    ↓
Controlled DOCX modification
```

Python should primarily provide:

* file I/O,
* document extraction,
* orchestration,
* state passing,
* edit application,
* output generation,
* logging.

Semantic decisions should remain with the LLM.

---

# Possible Future Framework

A graph/state-based framework such as LangGraph is a candidate for the final orchestration layer.

The workflow may eventually support:

* retry loops,
* alternative candidate investigation,
* reviewer rejection,
* human approval,
* confidence thresholds,
* audit logs.

However, a framework should not be introduced merely for the sake of using one.

The underlying workflow should be validated first.

---

# Next Experiment

The next immediate milestone is:

## End-to-End QTP Editing

Input:

```text
TRL_NEW.xlsx
+
OLD_QTP.docx
```

Output:

```text
UPDATED_QTP.docx
```

The system should:

1. identify required changes,
2. semantically locate QTP content,
3. generate exact edit instructions,
4. review those instructions,
5. apply the approved edits,
6. preserve unrelated content,
7. preserve document structure and formatting as far as possible.

At minimum test:

```text
T01
T03
T16
```

Only after this should the project move toward:

* 8B model comparison,
* larger documents,
* more adversarial benchmarks,
* LangGraph/CrewAI integration,
* Windows `.exe` packaging.

---

# Potential Final Windows Application

The eventual product could look conceptually like:

```text
+------------------------------------------+
|       QTP / TRL UPDATE AGENT             |
+------------------------------------------+
|                                          |
| TRL File:    [ Select .xlsx ]            |
| QTP File:    [ Select .docx ]            |
|                                          |
| Chapter scope: Chapter 4                 |
|                                          |
|          [ ANALYZE QTP ]                 |
|                                          |
+------------------------------------------+

                ↓

       Proposed Changes

T01  §4.4    UPDATE
T03  §4.9    UPDATE
T06  §4.10   UPDATE
T10  §4.16   NEEDS REVIEW
T16  §4.13   ADD

                ↓

       [ REVIEW CHANGES ]

                ↓

       [ GENERATE UPDATED QTP ]
```

The final application should maintain an audit trail showing:

* source TRL requirement,
* selected QTP location,
* evidence,
* proposed change,
* reviewer result,
* final decision,
* timestamp,
* output document.

---

# Current Status

| Area                          | Status                          |
| ----------------------------- | ------------------------------- |
| TRL parsing experiment        | Working                         |
| Synthetic change detection    | Working                         |
| Single-agent semantic mapping | Promising but vulnerable        |
| Multi-agent semantic mapping  | Promising                       |
| Contrarian review             | Demonstrated useful             |
| New requirement mapping       | Demonstrated                    |
| Exact edit planning           | Demonstrated on synthetic cases |
| Actual DOCX editing           | **Next milestone**              |
| Real QTP testing              | Not yet                         |
| 50-page real-world robustness | Not yet                         |
| 8B comparison                 | Later                           |
| Agent framework               | Later                           |
| Windows `.exe`                | Later                           |
| Production readiness          | Not established                 |

---

# Guiding Principle

The goal is not to build an impressive AI demo.

The goal is to determine, experimentally and honestly, whether a local LLM-based agent can perform enough of the real engineering-document reasoning to justify building a usable offline tool.

The system should be conservative:

**When evidence is strong → propose the change.**

**When evidence conflicts → investigate again.**

**When evidence is insufficient → NEEDS_REVIEW.**

**Never invent engineering requirements simply to produce an answer.**
