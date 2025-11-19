# Design Workbench - AI Agent Instructions

## Project Overview

This is an **Architecture Design Workbench** - a structured methodology for generating and critiquing comprehensive system architecture proposals. The workbench uses a template-driven, prompt-based workflow to help engineers systematically explore multiple architectural solutions for complex distributed systems.

**⚠️ CRITICAL: This is NOT a coding project.** This workbench is exclusively for design and architecture work. All outputs are markdown-based documentation (proposals, specifications, critiques). No code is written, compiled, or executed here.

## Architecture & Workspace Structure

### Core Directory Model
```
design-workbench/
├── sources/              # Input context files for AI prompts
├── templates/            # Reusable templates (prefix: _)
├── deliverables/         # Generated proposals & specs
├── temp/                 # Scratchpad for unfinished ideas
├── .github/
│   ├── prompts/         # AI prompt definitions
│   └── instructions/    # Project-specific AI guidance
└── examples/            # Reference implementations
    └── MyExampleProject/     # Complete example project
```

### The Source-Prompt-Deliverable Workflow

1. **Sources Directory** (`sources/`): Contains 5 core context files that define the problem space:
   - `overview.md` - Project purpose, goals, current system context, scope boundaries, and problem statement
   - `requirements.md` - Functional requirements, non-functional requirements, and mandatory design constraints
   - `facts.md` - Scale data, performance benchmarks, cost estimates, and system characteristics
   - `questions-to-answer.md` - Critical questions proposals must address
   - `glossary.md` - Key terms, concepts, and domain-specific vocabulary
   
   **Note**: The `temp/00_scratchpad.md` file serves as a scratchpad for unfinished ideas and is not automatically included in prompts. Use the IngestUpdates prompt to incorporate ideas from this file into the core source files and existing proposals when ready.
   
   **Reference Files**: The sources directory may also contain additional reference files (typically prefixed with `ZZ_`) for supplementary context. These files are loaded only when needed and referenced from the core context files. Examples include existing architecture specifications, performance analyses, or detailed technical documentation.

2. **Prompts Directory** (`.github/prompts/`): Contains specialized AI prompts:
   - `GenerateProposals.prompt.md` - Creates multiple high-level architecture proposals (one-pagers)
   - `RefineProposal.prompt.md` - Refines a selected high-level proposal into detailed, implementation-ready design
   - `CritiqueProposals.prompt.md` - Provides ruthless critique from Senior Principal Engineer perspective
   - `IngestUpdates.prompt.md` - Incrementally updates source files and existing proposals with new unstructured information

3. **Deliverables Directory**: AI-generated outputs organized by project:
   ```
   deliverables/[project-name]/
   ├── proposal-1-[pattern-name].md
   ├── proposal-2-[pattern-name].md
   ├── README.md                           # Executive summary & comparison
   └── supporting-documents/
       ├── architecture-spec-proposal-1.md  # Formal architecture spec
       ├── architecture-spec-proposal-2.md
       └── comparison-matrix.md             # Side-by-side evaluation
   ```

## Critical Workflow Patterns

### Starting a New Architecture Project

1. **Create source context files** from templates in `templates/`:
   ```powershell
   # Copy templates to sources/ and remove underscore prefix
   Copy-Item templates\_overview.md sources/overview.md
   # Repeat for all 5 source files
   ```

2. **Populate source files** with project-specific details:
   - `overview.md`: Define project purpose, goals, scope (in/out), current system context, and problem statement
   - `requirements.md`: List all functional requirements, non-functional requirements (SLAs, performance targets), and mandatory design constraints (technical, business, compliance)
   - `facts.md`: Add scale calculations, performance benchmarks, cost estimates, and known system characteristics
   - `questions-to-answer.md`: List critical questions each proposal must address
   - `glossary.md`: Define key terms, concepts, and domain-specific vocabulary

3. **Use the scratchpad for unfinished ideas**: The `temp/00_scratchpad.md` file can be used to collect rough notes and ideas before incorporating them via the IngestUpdates prompt

4. **Load the generation prompt**: Open `.github/prompts/GenerateProposals.prompt.md` and follow its instructions
   - The prompt will request all 5 source files as context
   - It will ask how many proposals to generate
   - It automatically creates the deliverables structure

### Ingesting Updates to Sources and Proposals

As you gather more information during the design process, use the **IngestUpdates** prompt to incrementally update both your source files and any existing proposals:

1. **Gather unstructured information**: Collect notes, meeting minutes, requirements, or feedback in any format
   - Can be freeform paragraphs
   - Bullet points
   - Email excerpts
   - Stakeholder feedback

2. **Load the update prompt**: Open `.github/prompts/IngestUpdates.prompt.md` and follow its instructions
   - Provide the unstructured information
   - The AI will break paragraphs into bullet points (if needed)
   - Each bullet point is classified into appropriate source file(s)
   - Source files are updated automatically with proper formatting
   - Existing proposals in deliverables/ are also updated with relevant changes

3. **Review the changes**: The AI provides a summary showing:
   - Which source files were updated
   - Which proposal documents were updated
   - What information was added to each file
   - Any architectural impacts or conflicts identified
   - Any classification decisions or duplicates skipped

**Benefits**:
- Eliminates manual classification of information
- Maintains consistency across source files and proposals
- Prevents duplicate entries
- Preserves existing organization and formatting
- Keeps proposals synchronized with evolving requirements and constraints
- Identifies architectural impacts early

### Generating Architecture Proposals

Once your source files are populated with sufficient context, use the **GenerateProposals** prompt to create comprehensive architecture proposals:

1. **Ensure source files are complete**: Verify all 5 core source files contain the necessary information
   - Review each file for completeness
   - Ensure requirements and constraints are clearly defined
   - Confirm scale data and facts are documented

2. **Load the generation prompt**: Open `.github/prompts/GenerateProposals.prompt.md` and follow its instructions
   - The AI will request all 5 core source files as context
   - Specify how many proposals you want (default is 2)
   - Optionally provide specific architectural patterns or approaches to explore

3. **Review generated high-level proposals**: The AI creates a concise proposal package:
   - Multiple one-page proposal documents with unique architectural approaches
   - Each proposal includes architecture diagram, component overview, and key trade-offs
   - Comparison matrix highlighting differences between approaches
   - README with executive summary and guidance
   - All files organized in `deliverables/[project-name]/`

**What the AI generates**:
- Each high-level proposal addresses all problems from `overview.md`
- Key trade-offs, benefits, and risks are clearly articulated
- Architecture diagrams show major components and interactions
- Sufficient detail for leadership to select promising approaches for further refinement

### Refining a High-Level Proposal into Detailed Design

After reviewing high-level proposals and selecting one (or more) for detailed design, use the **RefineProposal** prompt:

1. **Select a high-level proposal**: Choose which proposal from the deliverables folder to refine

2. **Load the refinement prompt**: Open `.github/prompts/RefineProposal.prompt.md` and follow its instructions
   - Provide all 5 core source files as context
   - Specify the path to the high-level proposal to refine
   - Optionally indicate specific areas to focus on (APIs, migration, operations, etc.)

3. **Review refined deliverables**: The AI transforms the high-level proposal into implementation-ready design:
   - Updated proposal document with detailed technical sections
   - Complete Architecture Specification document
   - Consolidated Architecture Decision Records (ADRs) document
   - Optional supporting documents (API specs, migration plans, operational runbooks)
   - All files organized in `deliverables/[project-name]/supporting-documents/`

**What the AI generates**:
- Detailed component specifications with interfaces and scaling characteristics
- Complete API definitions with request/response schemas
- Data models and storage architecture details
- Implementation roadmap with phased plan
- Migration strategy and operational runbooks
- Architecture Decision Records documenting all key decisions

### Critiquing Architecture Proposals

After proposals are generated (high-level or detailed), use the **CritiqueProposals** prompt to conduct expert-level critical analysis:

1. **Prepare for critique**: Gather the required materials
   - Source files (all 5 core files: overview, requirements, facts, questions-to-answer, glossary)
   - The specific proposal document(s) to be analyzed
   - Architecture specification(s) for the proposal(s) (if available for detailed proposals)

2. **Load the critique prompt**: Open `.github/prompts/CritiqueProposals.prompt.md` and follow its instructions
   - Provide all required source files
   - Specify which proposal(s) to critique
   - Optionally specify focus areas (performance, security, operations, etc.)
   - Indicate risk tolerance level (conservative vs. aggressive)

3. **Review the critique deliverables**: The AI generates comprehensive analysis:
   - Critical analysis document (`critique-[proposal-name].md`)
   - Failure scenario matrix
   - Performance impact analysis
   - Risk mitigation recommendations
   - All files created in the same directory as the proposal

**What the AI analyzes**:
- **Scale Feasibility**: Can the architecture handle required volume and throughput?
- **Requirements Compliance**: Does it meet all functional and non-functional requirements?
- **Failure Modes**: What can go wrong and what are the impacts?
- **Performance**: Where are the bottlenecks and latency issues?
- **Security**: What are the attack vectors and vulnerabilities?
- **Operations**: How complex is it to deploy, monitor, and maintain?

**Critique outputs help you**:
- Identify critical flaws before implementation
- Understand real-world failure scenarios
- Make informed go/no-go decisions
- Improve proposals through iteration
- Mitigate risks proactively

## Key Conventions

### File Naming
- **Templates**: Prefix with `_` (e.g., `_requirements.md`) - these are never used directly
- **Sources**: No prefix, lowercase with hyphens (e.g., `requirements.md`)
- **Proposals**: `proposal-[N]-[descriptive-pattern-name].md`
- **Specs**: `architecture-spec-proposal-[N].md`

## Maintaining Template and Prompt Synchronization

### Critical Requirement
This workbench relies on a strict correspondence between:
1. **Template files** in `templates/` (5 core files prefixed with `_`)
2. **Source file references** in this instruction document
3. **Source file references** in `.github/prompts/GenerateProposals.prompt.md`
4. **Source file references** in `.github/prompts/RefineProposal.prompt.md`
5. **Source file references** in `.github/prompts/CritiqueProposals.prompt.md`
6. **Source file references** in `.github/prompts/IngestUpdates.prompt.md`

### Current Template Inventory (5 Core Files)
The workbench currently uses these source context files:
1. `overview.md` - Project purpose, goals, current system context, scope boundaries, and problem statement
2. `requirements.md` - Functional requirements, non-functional requirements, and mandatory design constraints
3. `facts.md` - Scale data, performance benchmarks, cost estimates, and system characteristics
4. `questions-to-answer.md` - Critical questions proposals must address
5. `glossary.md` - Key terms, concepts, and domain-specific vocabulary

**Note**: The `temp/00_scratchpad.md` file serves as a scratchpad for unfinished ideas and is not automatically included in prompts. Use the IngestUpdates prompt to selectively incorporate ideas from this file into the core source files when ready.

### AI Agent Responsibilities

**When templates are added, removed, or renamed:**

1. **Update this instructions file** (`.github/copilot-instructions.md`):
   - Update the "Sources Directory" section listing all 5 files
   - Update the "Current Template Inventory" section in this "Maintaining Template and Prompt Synchronization" section
   - Update any numbered lists or step-by-step instructions that reference the file count
   - Ensure descriptions accurately reflect the purpose of each source file

2. **Update GenerateProposals prompt** (`.github/prompts/GenerateProposals.prompt.md`):
   - Update the "Project Context" section to include/remove file references
   - Update the "Prerequisites" section's numbered list of required source files
   - Ensure all file paths in markdown links are correct
   - Update the file count in any instructions (e.g., "provide the following X required source files")
   - Maintain consistency in file descriptions across all documents

3. **Update RefineProposal prompt** (`.github/prompts/RefineProposal.prompt.md`):
   - Update the "Project Context" section to include/remove file references
   - Update the "Prerequisites" section's numbered list of required source files
   - Ensure all file paths in markdown links are correct
   - Maintain consistency in file descriptions across all documents

4. **Update CritiqueProposals prompt** (`.github/prompts/CritiqueProposals.prompt.md`):
   - Update the "Prerequisites" section's numbered list
   - Ensure all source file references align with current templates
   - Update any analysis framework sections that reference specific source files

5. **Update IngestUpdates prompt** (`.github/prompts/IngestUpdates.prompt.md`):
   - Update the "Source Files Overview" section with the current list of core files
   - Update the classification table to reflect any new or changed source files
   - Ensure all file descriptions match those in other documents

6. **Validate consistency**:
   - Verify all four prompt documents reference the same set of source files
   - Confirm file descriptions are consistent across all documents
   - Ensure numbered sequences match the actual count of files
   - Test that all markdown links resolve correctly

### Synchronization Checklist

When making template changes, the AI agent must:
- [ ] List all template files in `templates/` directory
- [ ] Update the file count and list in `.github/copilot-instructions.md`
- [ ] Update the prerequisites section in `GenerateProposals.prompt.md`
- [ ] Update the prerequisites section in `RefineProposal.prompt.md`
- [ ] Update the prerequisites section in `CritiqueProposals.prompt.md`
- [ ] Update the source files overview in `IngestUpdates.prompt.md`
- [ ] Verify file descriptions match across all four documents
- [ ] Update any step-by-step instructions that reference file counts
- [ ] Confirm all markdown links are valid
- [ ] Report all changes made to the user for review
- [ ] Update the prerequisites section in `CritiqueProposals.prompt.md`
- [ ] Update the source files overview in `IngestUpdates.prompt.md`
- [ ] Verify file descriptions match across all four documents
- [ ] Update any step-by-step instructions that reference file counts
- [ ] Confirm all markdown links are valid
- [ ] Report all changes made to the user for review
