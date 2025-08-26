---
title: kai-eval-framework
authors:
  - "@savitharaghunathan"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2025-08-26
last-updated: 2025-08-26
status: provisional
see-also:
  - 
replaces:
  - 
superseded-by:
  - 
---

# Kai Evaluation Framework

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]

* How should we balance the cost of LLM-as-Judge evaluations?
* Extensibility to other langauges and frameworks
* How will we handle flaky tests or external CI failures skewing end-to-end metrics?

## Summary

The evaluation framework for Kai proposes a structured methodology to assess Kai’s performance at both the component and system levels. By defining clear metrics for agentic workflow, hints generation, and solution generation, alongside end-to-end system evaluation, the framework aims to quantify Kai’s effectiveness, accuracy, and efficiency in automating application migrations.

## Motivation

As Kai evolves, it is critical to demonstrate its value through measurable, repeatable evaluations. This enhancement addresses:

* Consistency: Standardizing evaluation across components and scenarios ensures results are comparable over time.
* Transparency: Clear metrics and result formats enable stakeholders to understand Kai’s strengths and limitations.
* Continuous Improvement: Feedback from eval drives targeted updates in Kai’s agents, hints generation, and solution generation.

### Goals

* Define and implement component-specific metrics for each Kai component.
* Establish an LLM-as-a-Judge prompt for evaluating generated solutions.
* Create end-to-end evaluation with defined metrics for migration time, test pass rate, manual intervention rate, and iteration count.
* Report evaluation results in a standardized template.

### Non-Goals

* Building a full UI dashboard for displaying evaluation data.
* Automating remediation based on evaluation outcomes.
* Defining new migration rules or modifying Kai's core migration logic.
* Replacing existing testing frameworks or CI/CD pipelines.

## Proposal

The evaluation framework will implement a two-tier assessment strategy:

 * Component-based evaluation: Test each Kai component in isolation to ensure individual functionality
 * End-to-end evaluation: Measure the integrated system's effectiveness across different parametrs

### User Stories

#### Story 1: Component Performance Assessment
As a Kai developer, I want to evaluate the functionality of individual components (agentic workflow, hints generation, solution generation) so that I can identify bottlenecks and optimize specific areas of the system.

#### Story 2: System Performance Comparison
As a Kai admin, I want to compare Kai's performance across different configurations (baseline, agentic, solution-server-only, solution server with agentic workflow) so that I can understand the value of each component and make informed deployment decisions.

#### Story 3: Quality Assurance
As a QA engineer, I want to run standardized evaluations on Kai's output so that I can ensure consistent quality across releases and identify regressions.

### Implementation Details/Notes/Constraints

* Component isolation: Each component will be evaluated independently to isolate performance characteristics and identify specific areas for improvement.

* LLM-as-Judge methodology*: For solution generation evaluation, we willl use a separate LLM prompt to review code based on established software engineering principles, since generated files cannot be compiled on its own.

* Standardized metrics: All evaluations will produce results in a consistent JSON format to enable automated analysis and comparison.

* Context preservation: The evaluation framework must maintain the original context (legacy code, incident details, solved examples) to ensure accurate assessment.

### Security, Risks, and Mitigations

* The LLM-as-Judge evaluation requires external API calls. We will implement rate limiting, API key rotation, and fallback mechanisms to mitigate API failures.

* Resource Consumption: e2e evaluations could consume significant computational resources. 

## Design Details

### Component-Based evaluation

#### 1. Agentic workflow 

Assess the agent's ability to autonomously manage the migration process.

**Metrics**:
- Task discovery accuracy: Percentage of correctly identified migration tasks from code analysis reports
- Resolution effectiveness: Success rate of agent-chosen actions in resolving identified issues
- Operational efficiency: Average time and number of steps to resolve a single issue

Evaluation method: Automated testing with predefined migration scenarios and expected outcomes.

#### 2. Hints Generation (RAG) Evaluation

Assess the quality of contextual hints provided to the solution generator.

Available GenAI frameworks: RAGAS, DeepEval, Promptfoo

**Metrics**:
- Faithfulness: Degree of factual consistency with provided context (1-5 scale)
- Context Relevance: Relevance of retrieved context to specific migration issues (1-5 scale)
- Answer Relevancy: Usefulness and actionability of generated hints (1-5 scale)

Evaluation method: Integration with established RAG evaluation frameworks using data obtained from scenarios

#### 3. Solution Generation Evaluation

Assess the quality of generated refactored code.

Evaluation method: LLM-as-Judge using CodeJudge principles. we use an LLM-as-Judge approach that follows established CodeJudge principles for evaluating code quality without execution. The following expert prompt template implements these principles

```yaml
export const LANGCHAIN_PROMPT_TEMPLATE = `
You are an expert software modernization architect with 15 years of experience, overseeing the migration of a large enterprise Java project from {source} to {target}. 

Your team uses the Konveyor code analysis tool to identify incidents that must be changed. Konveyor AI (Kai) is used to generate and apply the recommended changes. Your job is to meticulously review the changes made by Kai based *only* on the provided context.

It is critically important that Kai makes the minimum number of changes necessary to correct the problem identified by Konveyor and has avoided making unnecessary or superfluous changes. The AI may be deceptive; compare the original and changed files carefully.

### CONTEXT

1.  **Legacy Code Snippet:**
    \`\`\`java
    {{legacy_code}}
    \`\`\`

2.  **Incident Details (from Konveyor Analysis):**
    * **Rule ID:** {{rule_id}}
    * **Rule Description:** {{rule_description}}

3.  **Solved Example Provided (if any):**
    \`\`\`java
    // This is an example of how a similar problem was solved previously.
    {{solved_example}}
    \`\`\`

### KAI-GENERATED SOLUTION TO EVALUATE
\`\`\`java
{{generated_code}}
\`\`\`

### EVALUATION INSTRUCTIONS

First, perform a step-by-step analysis by following the four steps below. Write down your reasoning for each step. After completing your analysis, provide a final verdict in the specified JSON format. The JSON object MUST be the only content in your response.

#### Step 1: Semantic Equivalence Analysis
Analyze if the generated solution preserves the exact functional behavior of the legacy code. Note any potential regressions or changes in logic.

*Reasoning:*


#### Step 2: Rule Adherence Analysis
Determine if the generated solution directly and completely addresses the issue described in the 'Incident Details'. This evaluates how well the changes match the recommendations made by Konveyor.

*Reasoning:*


#### Step 3: Faithfulness to Context Analysis
Check if the solution correctly uses patterns from the 'Solved Example' (if provided) and avoids introducing any new logic, libraries, or hardcoded values not supported by the provided context.

*Reasoning:*


#### Step 4: Code Quality Analysis
Assess the generated code for readability, maintainability, and adherence to best practices for Java and the {target} framework. Also, confirm that only the minimum necessary changes were made.

*Reasoning:*


-----

### FINAL VERDICT
**IMPORTANT:** The generated solution must be evaluated as a whole. If it is not a syntactically complete file or contains unfinished method bodies, all scores must be 0 and 'is_compilable' must be false.

\`\`\`json
{
  "semantic_equivalence_score": "<Provide a single integer score from 1-5, where 1=Incorrect/Major Issues and 5=Perfectly Equivalent>",
  "rule_adherence_score": "<Provide a single integer score from 1-5, where 1=Rule Ignored and 5=Rule Perfectly Adhered To>",
  "faithfulness_score": "<Provide a single integer score from 1-5, where 1=Hallucinated/Contradicted Context and 5=Perfectly Faithful to Context>",
  "code_quality_score": "<Provide a single integer score from 1-5, where 1=Poor Quality/Many Unnecessary Changes and 5=Excellent Quality/Minimal Changes>",
  "is_compilable": "<Provide a boolean value (true/false) indicating if the code appears syntactically valid and complete>",
  "detailed_notes": "<Provide a concise summary of your reasoning for the final scores, referencing your step-by-step analysis.>"
}
\`\`\`
`;

```

**Eval criteria**:
- Semantic Equivalence Score: Preservation of functional behavior (1-5 scale)
- Rule Adherence Score: Compliance with Konveyor recommendations (1-5 scale)
- Faithfulness Score: Adherence to provided context (1-5 scale)
- Code Quality Score: Readability and maintainability (1-5 scale)
- Compilability: Boolean indicating syntactic validity

### End-to-End (E2E) System Evaluation

Evaluate Kai's overall effectiveness by running the full migration workflow on sample applications.

**Scenarios**:
1. **Baseline**: Agents off, Solution Server off (RAG hints disabled)
2. **Agents Only**: Agents on, Solution Server off
3. **Solution Server Only**: Agents off, Solution Server on (RAG hints enabled)
4. **Full Kai**: Agents on, Solution Server on

**Metrics**:
- **Total Migration Time**: Time from start to end of automated migration process
- **Code Quality & Correctness**: Final pass rate of project's unit and integration test suite
- **Manual Intervention Rate**: Number of times human intervention is required
- **Iteration Count**: Number of attempts to achieve stable migration

Automated migration of standardized test applications with pre-defined test suites and performance monitoring. successfully run of the tests would show that the migration is succesful

### Test Plan
<todo>
### Upgrade / Downgrade Strategy

<todo>

## Implementation History

- Phase 1: Component evaluation framework implementation
- Phase 2: LLM-as-Judge integration and testing
- Phase 3: End-to-end evaluation scenarios
- Phase 4: Performance optimization and metric refinement
- Phase 5: Production deployment and monitoring

## Drawbacks
<todo>

## Alternatives

* Human-only evaluation would eliminate automation costs but reduce consistency and scalability.

* Using fewer, simpler metrics would reduce complexity but provide less granular insights.


## Infrastructure Needed

**Infrastructure**:
- CI/CD pipeline integration for automated evaluation
- Metrics collection and storage system
- Integration with RAG evaluation frameworks (RAGAS, DeepEval, Promptfoo)
- LLM API access for judge evaluations
- Test application repositories for E2E evaluation

