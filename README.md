# Architecture Design Workbench

A structured approach to generating and evaluating system architecture proposals using AI-assisted prompts.

## What This Does

This workbench helps you systematically design complex distributed systems by:
- Generating multiple architecture proposals from your requirements
- Refining high-level proposals into detailed implementation specs
- Critiquing designs to identify risks and failure modes
- Keeping your design documents synchronized as requirements evolve

## Available Prompts

Four specialized prompts guide your design process:

1. **GenerateProposals** - Creates multiple architecture options (high-level one-pagers) from your requirements
2. **RefineProposal** - Expands a selected proposal into detailed implementation-ready specs with ADRs
3. **CritiqueProposals** - Provides expert-level critique identifying risks, bottlenecks, and failure scenarios
4. **IngestUpdates** - Updates your source files and proposals with new information as requirements change

## Quick Start

### 1. Set Up Your Project Context

Copy the templates to create your source files:

```powershell
Copy-Item templates\_overview.md sources/overview.md
Copy-Item templates\_requirements.md sources/requirements.md
Copy-Item templates\_facts.md sources/facts.md
Copy-Item templates\_questions-to-answer.md sources/questions-to-answer.md
Copy-Item templates\_glossary.md sources/glossary.md
```

Fill in each file with your project details:
- **overview.md** - What you're building and why
- **requirements.md** - Functional and non-functional requirements
- **facts.md** - Scale numbers, performance targets, cost constraints
- **questions-to-answer.md** - Critical questions proposals must address
- **glossary.md** - Key terms and concepts

### 2. Generate Proposals

Open `.github/prompts/GenerateProposals.prompt.md` in Copilot Chat and follow the instructions. The AI will ask for your source files and create multiple architecture proposals.

### 3. Refine Your Chosen Approach

Select a promising proposal and open `.github/prompts/RefineProposal.prompt.md`. The AI will expand it into a detailed architecture spec with component designs, API definitions, and implementation plans.

### 4. Critique Before Building

Open `.github/prompts/CritiqueProposals.prompt.md` to get expert-level analysis of risks, failure modes, and bottlenecks. Use this feedback to iterate on your design.

### 5. Keep Things Updated

As requirements change, use `.github/prompts/IngestUpdates.prompt.md` to automatically update your source files and existing proposals with new information.

## Typical Workflow

```
Define Context → Generate Options → Select & Refine → Critique → Iterate
     ↓               ↓                   ↓              ↓          ↓
  sources/      deliverables/      spec details    risks found   update
```

1. Fill in your `sources/` files with requirements and constraints
2. Generate 2-3 high-level proposals to explore different approaches
3. Refine your preferred proposal into detailed specs
4. Run critique to identify issues early
5. Update documents as you learn more (IngestUpdates)

**Key Points:**
- **IngestUpdates can happen at any time** - Use it whenever new information emerges to keep all documents synchronized with the latest changes
- **Use proposals as inspiration** - Think of this as working with an expert architect throughout your design process. Refine the proposals as needed, discarding unrealistic assumptions and vetting design decisions using the IngestUpdates prompt
- **⚠️ AI-Generated Content Disclaimer** - Generated proposals and critiques may contain inaccurate details, as with any AI-generated response. Always validate against your specific requirements and constraints

## Output Structure

All generated proposals and specs go into `deliverables/[your-project]/`:
- `proposal-N-[name].md` - High-level architecture proposals
- `supporting-documents/` - Detailed specs, ADRs, and critiques

## Examples

Check `examples/` for complete reference implementations showing the full workflow.

---

**Remember**: This workbench is for design work only. All outputs are markdown documentation, not code.
