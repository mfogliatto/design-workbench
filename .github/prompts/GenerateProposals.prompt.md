# Architecture Design Proposals - Generation Prompt

## Overview

### Purpose
You are a Software Engineer expert in cloud computing and Distributed Systems, specialized in high-throughput, low-latency systems. Your task is to generate comprehensive design proposals for the system architecture that addresses the critical architectural problems while maintaining operational excellence.

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

## High-Level Proposal Requirements Framework

Each high-level architectural proposal must be a concise one-pager that includes:

### 1. Executive Summary
- **Problem Statement**: Brief recap of the key architectural challenges being addressed
- **Proposed Solution**: High-level description of the architectural approach (2-3 paragraphs)
- **Key Benefits**: Top 3-5 benefits this architecture provides over current state

### 2. High-Level Architecture
- **Architecture Diagram**: Visual representation showing major components and their interactions
- **Component Overview**: Brief description of each major component and its primary responsibility
- **Data Flow**: High-level description of how data moves through the system
- **Integration Points**: Key external dependencies and integration patterns

### 3. Core Trade-offs
- **Key Advantages**: Top 3-4 strengths of this approach
- **Key Disadvantages**: Top 3-4 limitations or concerns with this approach
- **Risk Summary**: Highest risks and general mitigation approach

### 4. Feasibility Assessment
- **Implementation Complexity**: High/Medium/Low with brief justification
- **Operational Impact**: Expected changes to operational practices
- **Migration Approach**: High-level strategy for transitioning from current state
- **Rough Order of Magnitude**: Preliminary estimate of effort and cost (if sufficient data available)

### 5. Cost Model
Each proposal must include a standardized cost model table to enable direct cost comparison across proposals. The cost model must follow this structure:

| Cost Factor | Unit | Volume/Usage | Unit Cost | Monthly Cost | Annual Cost | Notes |
|-------------|------|--------------|-----------|--------------|-------------|-------|
| **Compute** | | | | | | |
| - Service/VM instances | instance-hours | [hours/month] | $[X]/hour | $[total] | $[total] | [instance type, size] |
| - Serverless compute | million requests | [requests/month] | $[X]/1M requests | $[total] | $[total] | [function type, memory] |
| - Container instances | vCPU-hours | [hours/month] | $[X]/vCPU-hour | $[total] | $[total] | [container specs] |
| **Storage** | | | | | | |
| - Primary storage | GB/month | [GB] | $[X]/GB | $[total] | $[total] | [storage type, tier] |
| - Storage read operations | million ops | [ops/month] | $[X]/1M ops | $[total] | $[total] | [operation type] |
| - Storage write operations | million ops | [ops/month] | $[X]/1M ops | $[total] | $[total] | [operation type] |
| **Network** | | | | | | |
| - Data transfer (egress) | GB | [GB/month] | $[X]/GB | $[total] | $[total] | [regional/cross-region] |
| - Data transfer (ingress) | GB | [GB/month] | $[X]/GB | $[total] | $[total] | [usually free] |
| - Load balancer | LCU-hours | [hours/month] | $[X]/LCU-hour | $[total] | $[total] | [LB type] |
| **Message/Event Services** | | | | | | |
| - Message operations | million ops | [ops/month] | $[X]/1M ops | $[total] | $[total] | [queue/topic operations] |
| - Event streaming | GB | [GB/month] | $[X]/GB | $[total] | $[total] | [partition units] |
| **Database** | | | | | | |
| - Database instances | instance-hours | [hours/month] | $[X]/hour | $[total] | $[total] | [DB type, size] |
| - Database storage | GB/month | [GB] | $[X]/GB | $[total] | $[total] | [IOPS tier] |
| - Database read operations | million ops | [ops/month] | $[X]/1M ops | $[total] | $[total] | [RU/s, query type] |
| - Database write operations | million ops | [ops/month] | $[X]/1M ops | $[total] | $[total] | [RU/s, operation type] |
| **Observability** | | | | | | |
| - Logs ingestion | GB | [GB/month] | $[X]/GB | $[total] | $[total] | [retention period] |
| - Metrics/traces | data points | [points/month] | $[X]/1M points | $[total] | $[total] | [cardinality] |
| - Alerts/notifications | alerts | [alerts/month] | $[X]/alert | $[total] | $[total] | [notification channels] |
| **Other Services** | | | | | | |
| - [Service name] | [unit] | [volume] | $[X]/[unit] | $[total] | $[total] | [description] |
| **TOTAL** | | | | **$[total]** | **$[total]** | |

**Cost Model Instructions**:
- Include only rows relevant to the specific proposal architecture
- Use standardized units for comparability across proposals
- Base estimates on volumes from `facts.md` or provide assumptions
- Include notes on pricing tier, region, or specific service details
- If exact costs are unknown, provide reasonable estimates with assumptions documented
- For new services not in current state, research current market pricing
- Consider reserved instance discounts vs. on-demand pricing where applicable

**Note**: For detailed, implementation-ready designs with complete technical specifications, Architecture Specification documents, and Architecture Decision Records (ADRs), use the `RefineProposal.prompt.md` to refine a high-level proposal after it has been validated.

---

## Task Definition

### Prerequisites
**BEFORE PROCEEDING**: You must first ask the user to provide the following required source files from the `../../sources/` directory:

1. **`overview.md`** - Ask for the project purpose, goals, current system context, scope boundaries, and problem statement
2. **`requirements.md`** - Gather functional requirements, non-functional requirements, and mandatory design constraints
3. **`facts.md`** - Gather scale calculations, performance benchmarks, cost estimates, and known system characteristics
4. **`questions-to-answer.md`** - Critical questions that each design proposal must address
5. **`glossary.md`** - Request definitions of key terms, concepts, and domain-specific vocabulary

**Additionally, ask the user**:
6. **Number of proposals** - How many architectural design proposals they want you to generate (default to **two** if not specified)
7. **Additional guidance or ideas** - Any specific architectural patterns, technologies, approaches, or constraints they want you to consider or explore when generating the proposals (e.g., "Consider event-driven patterns", "Explore options with minimal new infrastructure", "Include a serverless approach", etc.)

**Note**: This prompt generates **high-level proposals only** (one-pagers). For detailed, implementation-ready designs, first use this prompt to generate and validate high-level proposals, then use `RefineProposal.prompt.md` to develop a selected proposal into a comprehensive detailed design.

**IMPORTANT**: If these files do not exist, ask the user if they would like you to help generate them based on information they can provide about their project. Do not proceed with proposal generation until you have access to this foundational information.

### Goal
Generate **the requested number of high-level architectural design proposals** (one-pagers) that provide sufficient clarity for technical leadership to understand different architectural approaches and select the most promising direction for further detailed design.

### Success Criteria
Each high-level proposal must fulfill the following criteria:
- Clearly articulates the core architectural approach and how it addresses the identified problems
- Provides a visual architecture diagram showing major components and interactions
- Highlights key trade-offs, benefits, and risks in a concise manner
- Enables informed decision-making about which approach(es) warrant detailed design investment
- Can be read and understood in 5-10 minutes by technical stakeholders

### Deliverables
**File Organization**: All generated proposals must be created in the `../../deliverables/` directory within a project-specific subfolder. The subfolder name should clearly identify the project for which these proposals are being created.

**Folder Structure**:
```
../../deliverables/
└── [project-name]/
    ├── proposal-1-[descriptive-name].md
    ├── proposal-2-[descriptive-name].md
    ├── proposal-N-[descriptive-name].md  (for additional proposals)
    ├── README.md
    └── supporting-documents/
        └── comparison-matrix.md
```

**Required Outputs**:
1. **Generate the requested number of high-level proposals**: Each should be a concise one-pager with a descriptive name reflecting its unique architectural pattern or approach. Each proposal must include the standardized cost model table defined in section 5 of the High-Level Proposal Requirements Framework.
2. **Comparison Matrix**: A side-by-side comparison of all generated proposals highlighting key differences in approach, complexity, risk, and benefits. The comparison matrix must include a consolidated cost comparison section showing total monthly and annual costs for each proposal, along with key cost drivers and trade-offs.
3. **README**: Executive summary explaining the project context and providing guidance on which proposals to consider for different scenarios

**Next Steps**:
After high-level proposals are reviewed and validated, use `RefineProposal.prompt.md` to develop selected proposals into detailed, implementation-ready designs with complete technical specifications, Architecture Specification documents, and Architecture Decision Records (ADRs).

---
