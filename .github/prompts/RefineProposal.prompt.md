# Architecture Proposal Refinement - Detailed Design Prompt

## Overview

### Purpose
You are a Software Engineer expert in cloud computing and Distributed Systems, specialized in high-throughput, low-latency systems. Your task is to take an existing high-level architecture proposal and refine it into a comprehensive, implementation-ready detailed design with complete technical specifications.

## Project Context
**See the project purpose, goals, current system context, scope boundaries, and problem statement**: [Overview](../../sources/overview.md)

---

## Requirements & Constraints

**See detailed requirements and mandatory design constraints**: [System Requirements](../../sources/requirements.md)

---

## Facts

**See scale estimates, performance benchmarks, cost data, and system characteristics**: [Facts](../../sources/facts.md)

---

## Questions to Answer

**See critical questions that each design proposal must address**: [Questions to Answer](../../sources/questions-to-answer.md)

---

## Glossary

**See definitions of key terms and concepts**: [Glossary](../../sources/glossary.md)

---

## Detailed Design Requirements Framework

The refined proposal must include all sections from the high-level proposal plus:

### 1. Detailed Component Specifications
- **Complete component breakdown** with purpose, interfaces, and scaling characteristics for each component
- **API specifications** with detailed endpoint definitions, request/response schemas, and error handling
- **Data models** including schemas for all entities, storage structures, and data flows
- **Integration patterns** with detailed sequence diagrams and interaction protocols
- **Storage architecture** with partitioning strategies, replication models, and consistency guarantees

### 2. Architecture Specification Document
- **Complete Architecture Specification**: A fully completed copy of the provided architecture specification template with all applicable sections filled in for the proposed design
- **Technical Implementation Details**: All sections of the template must be thoroughly documented with proposal-specific details
- **Service Dependencies**: Clear documentation of upstream/downstream dependencies and integration points
- **Deployment Models**: Detailed deployment topology, scaling policies, and resource allocation strategies
- **Monitoring & Observability**: Comprehensive metrics, logging strategies, and alerting definitions

### 3. Architecture Decision Records (ADRs)
- **Consolidated ADR Document**: A single comprehensive document containing all Architecture Decision Records (ADRs) for key architectural decisions made in the proposal
- **ADR Structure**: Each ADR within the document must include:
  - **Title & Status**: Clear title and current status (Proposed, Accepted, Deprecated, Superseded)
  - **Context**: Explanation of the problem or situation requiring a decision, including relevant constraints and requirements
  - **Decision**: The chosen approach with detailed rationale explaining why this option was selected
  - **Consequences**: Both positive impacts (benefits gained) and negative impacts (trade-offs, limitations, risks introduced)
  - **Alternatives Considered**: Other viable options that were evaluated and specific reasons why they were rejected
- **Coverage**: ADRs should document significant decisions such as architectural patterns, technology choices, integration strategies, data flow approaches, scaling strategies, and cross-cutting concerns
- **Traceability**: Each ADR should reference relevant requirements, constraints, or problems from the source documents that influenced the decision

### 4. Implementation Roadmap
- **Phased implementation plan** with clear milestones and dependencies
- **Migration strategy** detailing how to transition from current system to proposed architecture
- **Testing strategy** including unit, integration, performance, and chaos testing approaches
- **Rollout plan** with feature flags, canary deployments, and rollback procedures

### 5. Operational Excellence
- **Runbook entries** for common operational scenarios
- **Incident response procedures** for failure modes identified in the design
- **Capacity planning** with growth projections and scaling triggers
- **Cost modeling** with detailed TCO analysis and optimization opportunities

---

## Task Definition

### Prerequisites
**BEFORE PROCEEDING**: You must first ask the user to provide:

1. **Source files** from the `../../sources/` directory:
   - `overview.md` - Project purpose, goals, current system context, scope boundaries, and problem statement
   - `requirements.md` - Functional requirements, non-functional requirements, and mandatory design constraints
   - `facts.md` - Scale calculations, performance benchmarks, cost estimates, and known system characteristics
   - `questions-to-answer.md` - Critical questions that the design must address
   - `glossary.md` - Definitions of key terms, concepts, and domain-specific vocabulary

2. **Architecture Specification Template**:
   - Path to the architecture specification template file (e.g., `../../sources/architecture-spec.md` or `../../templates/_architecture-spec.md`)
   - This template will be used to generate the complete Architecture Specification document for the proposal

3. **High-level proposal** to refine:
   - Path to the existing high-level proposal document (e.g., `../../deliverables/[project-name]/proposal-1-[name].md`)
   - Confirm which proposal from the deliverables folder should be refined

4. **Refinement focus areas** (optional):
   - Any specific areas the user wants you to focus on or expand (e.g., "Detail the API contracts", "Elaborate on the migration strategy", "Provide detailed cost breakdown")
   - Any new requirements or constraints that have emerged since the high-level proposal was created

### Goal
Transform the high-level architectural proposal into a comprehensive, implementation-ready detailed design with sufficient technical depth for engineering teams to begin building the system.

### Success Criteria
The refined proposal must:
- Maintain all key architectural decisions from the high-level proposal
- Provide implementation-ready technical specifications for all components
- Include complete API definitions and data models
- Address all technical questions that would arise during implementation planning
- Enable engineering teams to create accurate time and effort estimates
- Provide clear operational playbooks and monitoring strategies

### Deliverables
**File Organization**: All generated detailed design documents will be created in the same directory as the original high-level proposal within `../../deliverables/[project-name]/`.

**Output Structure**:
```
../../deliverables/[project-name]/
├── proposal-N-[descriptive-name].md  (updated with detailed sections)
└── supporting-documents/
    ├── architecture-spec-proposal-N.md  (newly created)
    ├── adrs-proposal-N.md  (newly created)
    ├── detailed-component-specs-proposal-N.md  (optional, if needed)
    ├── api-specifications-proposal-N.md  (optional, if needed)
    ├── migration-plan-proposal-N.md  (optional, if needed)
    └── operational-runbook-proposal-N.md  (optional, if needed)
```

**Required Outputs**:
1. **Updated Proposal Document**: The original high-level proposal expanded with detailed technical sections
2. **Architecture Specification**: Complete architecture specification document following the template
3. **Architecture Decision Records**: Consolidated ADR document with all significant architectural decisions
4. **Optional Supporting Documents**: Additional detailed design documents as needed based on complexity and scope

---

## Execution Instructions

1. **Load Context**: Read and understand the source files and the existing high-level proposal
2. **Identify Gaps**: Determine what detailed information needs to be added to make the proposal implementation-ready
3. **Generate Details**: Create comprehensive technical specifications for each component and interface
4. **Document Decisions**: Capture all architectural decisions in ADRs with full context and alternatives considered
5. **Complete Specification**: Fill out the complete Architecture Specification Template with all technical details
6. **Validate Completeness**: Ensure the refined proposal addresses all requirements and provides sufficient detail for implementation

---
