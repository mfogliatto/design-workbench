# Architecture Design Proposals - Critical Analysis Prompt

## Overview

### Purpose
You are a Software Engineer expert in large-scale distributed systems, performance engineering, and operational reliability. Your task is to conduct a comprehensive, ruthless critique of proposed system architectures, identifying critical flaws, failure scenarios, and operational challenges that could lead to system failures, SLA violations, or production incidents.

## Project Context
**See the project purpose, goals, current system context, scope boundaries, and problem statement**: [Overview](../../sources/overview.md)

---

## Analysis Framework

### Core Analysis Areas

#### 1. Scale Feasibility Analysis
**Examine whether the proposed architecture can handle the specified scale requirements:**

**Scale Requirements Reference**: [Facts - Scale & Performance](../../sources/facts.md)

**Analysis Focus**:
- **Volume Capacity**: Can the architecture handle the specified message/transaction volume?
- **Data Throughput**: Will the system support the required data processing rates?
- **Resource Requirements**: What infrastructure resources are needed and are they realistic?
- **Performance Degradation**: How does performance change under load?
- **Scaling Bottlenecks**: What components become bottlenecks at scale?

**Key Questions**:
- What happens when traffic increases 2x, 5x, 10x above normal?
- Which components will fail first under extreme load?
- How much infrastructure is required to meet peak demand?
- What are the network bandwidth requirements and limitations?

#### 2. Requirements Compliance Validation
**Verify that the proposal actually addresses all stated requirements:**

**Requirements & Constraints Reference**: [System Requirements](../../sources/requirements.md)
**Problem Statement Reference**: [Overview](../../sources/overview.md)

**Analysis Focus**:
- **Functional Requirements**: Does the architecture support all required functionality?
- **Non-Functional Requirements**: Are performance, security, reliability targets achievable?
- **Design Constraints**: Does the proposal comply with all mandatory constraints?
- **Implicit Requirements**: Are there unstated requirements that must be met?

**Key Questions**:
- Which requirements are not adequately addressed?
- What assumptions about requirements might be incorrect?
- How will requirement conflicts be resolved?
- What new requirements does this architecture introduce?

#### 3. Failure Mode Analysis
**Identify comprehensive failure scenarios and their impacts:**

**Analysis Focus**:
- **Single Points of Failure**: What components can cause total system failure?
- **Cascading Failures**: How do component failures propagate through the system?
- **Network Partitions**: How does the system behave during connectivity issues?
- **Resource Exhaustion**: What happens when CPU, memory, or network resources are depleted?
- **Data Corruption**: How might data integrity be compromised?

**Failure Categories to Examine**:
- **Hardware Failures**: Server, network, storage failures
- **Software Failures**: Bugs, memory leaks, crashes, deadlocks
- **Network Failures**: Partitions, latency spikes, bandwidth exhaustion
- **Human Errors**: Misconfigurations, deployment mistakes, operational errors
- **External Dependencies**: Third-party service failures, API rate limits
- **Security Incidents**: Attacks, breaches, privilege escalations

#### 4. Performance and Latency Analysis
**Evaluate performance characteristics and identify bottlenecks:**

**Performance SLA Reference**: [Facts - Performance Benchmarks](../../sources/facts.md)

**Analysis Focus**:
- **Latency Budgets**: How much latency does each component contribute?
- **Throughput Limits**: What are the maximum processing rates for each component?
- **Resource Utilization**: How efficiently does the architecture use resources?
- **Serialization Overhead**: What are the costs of data marshaling/unmarshaling?
- **Network Overhead**: How much additional network traffic is generated?

**Performance Anti-Patterns to Identify**:
- **N+1 Query Problems**: Excessive database or service calls
- **Chatty Interfaces**: Too many small network calls
- **Synchronous Bottlenecks**: Blocking operations that limit throughput
- **Resource Contention**: Components competing for shared resources
- **Inefficient Serialization**: Expensive data format conversions

#### 5. Security Vulnerability Assessment
**Analyze security implications and attack vectors:**

**Security Requirements Reference**: [System Requirements - Security](../../sources/requirements.md)

**Analysis Focus**:
- **Attack Surface**: How does the architecture expand potential attack vectors?
- **Data Protection**: How is sensitive data protected in transit and at rest?
- **Authentication/Authorization**: How are services authenticated and access controlled?
- **Privilege Escalation**: Can compromised components access more than intended?
- **Audit Trail**: How are security events tracked and monitored?

**Security Threats to Consider**:
- **Man-in-the-Middle**: Interception of inter-service communication
- **Service Spoofing**: Malicious services impersonating legitimate ones
- **Data Exfiltration**: Unauthorized access to sensitive information
- **Denial of Service**: Attacks that overwhelm system resources
- **Privilege Escalation**: Gaining higher permissions than intended

#### 6. Operational Complexity Assessment
**Evaluate the operational burden and maintenance challenges:**

**Analysis Focus**:
- **Monitoring Complexity**: How difficult is it to monitor system health?
- **Deployment Risks**: What can go wrong during deployments?
- **Debugging Difficulty**: How hard is it to troubleshoot issues?
- **Maintenance Overhead**: What ongoing maintenance is required?
- **Skill Requirements**: What expertise is needed to operate the system?

**Operational Anti-Patterns to Identify**:
- **Monitoring Explosion**: Too many metrics and alerts to manage effectively
- **Deployment Coupling**: Services that must be deployed together
- **Debug Complexity**: Distributed systems that are hard to troubleshoot
- **Manual Processes**: Operations that require frequent human intervention
- **Skills Gap**: Architecture requiring expertise not available in the organization

---

## Analysis Methodology

### Prerequisites
**BEFORE PROCEEDING**: You must first gather the following required source files:

1. **`overview.md`** - Understanding the business context, current system, scope, and problem statement
2. **`requirements.md`** - Functional requirements, non-functional requirements, and mandatory design constraints
3. **`facts.md`** - Scale estimates, performance benchmarks, cost data, and system characteristics
4. **`questions-to-answer.md`** - Critical questions proposals must address
5. **`glossary.md`** - Key terms and domain-specific vocabulary

**Additionally, ask the user to provide**:
7. **The specific proposal document(s)** to be analyzed
8. **Focus areas** - Which aspects they want emphasized in the critique (if any)
9. **Risk tolerance** - Whether they want conservative or aggressive risk assessment

### Analysis Process

#### Step 1: Comprehensive Document Review
- Read and understand all provided documentation thoroughly
- Identify any gaps, inconsistencies, or unclear specifications
- Note assumptions made by the proposal authors
- Cross-reference claims against requirements and constraints

#### Step 2: Scale Impact Analysis  
- Calculate resource requirements based on provided scale estimates
- Model performance under normal, peak, and extreme load conditions
- Identify scaling bottlenecks and resource limitations
- Estimate infrastructure costs and operational overhead

#### Step 3: Failure Scenario Development
- Create detailed failure scenarios based on real-world incident patterns
- Analyze cascading failure possibilities and blast radius
- Evaluate recovery time objectives and procedures
- Assess data loss risks and business continuity impacts

#### Step 4: Comparative Analysis
- Compare proposed architecture against current architecture (if described)
- Identify what problems are solved vs. new problems introduced
- Evaluate whether benefits justify the costs and risks
- Consider alternative approaches that might be less risky

#### Step 5: Risk Assessment and Recommendations
- Categorize risks by severity and probability
- Provide specific recommendations for risk mitigation
- Suggest alternative approaches or architectural modifications
- Propose validation steps to test assumptions before implementation

---

## Deliverable Requirements

### File Organization
**All critique documents must be created in the same directory as the proposal being analyzed.**

### Required Outputs

#### 1. Critical Analysis Document
**Filename**: `critique-[proposal-name].md`

**Required Sections**:
- **Executive Summary**: High-level assessment of risks and recommendations
- **Methodology**: How the analysis was conducted
- **Critical Design Flaws**: Major architectural problems and vulnerabilities
- **Scale-Specific Failure Analysis**: Detailed failure scenarios at required scale
- **Security and Compliance Vulnerabilities**: Security risks and regulatory concerns  
- **Operational Complexity Assessment**: Day-to-day operational challenges
- **Alternative Architecture Recommendations**: Suggestions for better approaches
- **Conclusion and Risk Assessment**: Overall risk rating and final recommendations

#### 2. Failure Scenario Matrix
**Content**: Tabular analysis of failure modes, their probability, impact, and mitigation strategies

#### 3. Performance Impact Analysis
**Content**: Detailed performance modeling showing latency contributions and throughput limits

#### 4. Risk Mitigation Recommendations
**Content**: Specific, actionable recommendations for addressing identified risks

---

## Analysis Standards

### Criticism Guidelines

#### Be Ruthlessly Objective
- **Challenge Every Assumption**: Question all unstated assumptions and design decisions
- **Assume Murphy's Law**: If something can go wrong, it will - plan accordingly  
- **Real-World Focus**: Base analysis on actual production incident patterns
- **Scale Reality**: Consider what actually happens at massive scale, not theoretical ideals
- **No Sacred Cows**: Criticize any aspect that presents risks, regardless of architectural fashion

#### Provide Constructive Alternatives
- **Don't Just Criticize**: Offer concrete alternatives and improvements
- **Risk vs. Benefit**: Weigh risks against benefits fairly
- **Implementation Reality**: Consider practical constraints and migration challenges
- **Business Context**: Understand business drivers and constraints behind architectural choices

#### Use Concrete Examples
- **Specific Scenarios**: Provide detailed failure scenarios with timelines and impacts
- **Real Numbers**: Use actual scale figures and performance calculations
- **Historical Context**: Reference similar system failures and lessons learned
- **Quantified Risk**: Provide specific risk estimates where possible

### Analysis Depth Requirements

#### Surface-Level Issues (Always Include)
- Obvious architectural flaws and anti-patterns
- Clear performance bottlenecks and scalability limits
- Basic security vulnerabilities and compliance gaps
- Simple operational complexity issues

#### Deep Analysis (Required for High-Risk Systems)
- Subtle race conditions and distributed systems edge cases
- Complex failure mode interactions and cascading effects
- Advanced security attack vectors and privilege escalation paths
- Sophisticated operational scenarios and human error possibilities

#### Expert-Level Insights (Required for Mission-Critical Systems)
- Non-obvious failure modes based on production experience
- Performance characteristics under extreme load conditions
- Security implications of complex architectural interactions
- Long-term maintenance and evolution challenges

---

## Success Criteria

### Analysis Quality Metrics
- **Completeness**: All major risk categories thoroughly addressed
- **Specificity**: Concrete examples and scenarios provided for each risk
- **Actionability**: Clear recommendations for addressing identified issues
- **Realism**: Analysis grounded in real-world constraints and experiences
- **Balance**: Fair assessment of both risks and benefits

### Outcome Expectations
- **Risk-Informed Decision**: Enable stakeholders to make informed go/no-go decisions
- **Improved Design**: Provide input for architectural improvements and iterations
- **Risk Mitigation**: Identify specific steps to reduce identified risks
- **Alternative Evaluation**: Compare current proposal against other potential approaches

### Deliverable Standards
- **Professional Quality**: Documents suitable for executive and engineering review
- **Technical Depth**: Sufficient detail for engineering teams to understand and act on findings
- **Clear Communication**: Accessible to both technical and business stakeholders
- **Comprehensive Coverage**: All identified risks and alternatives thoroughly documented

---

## Task Definition

### Goal
Conduct a comprehensive, expert-level critique of the provided architectural proposal, identifying critical risks, failure scenarios, and operational challenges that could impact system reliability, performance, security, or business objectives.

### Success Metrics
- Identification of all major architectural risks and failure modes
- Detailed analysis of scalability and performance implications  
- Comprehensive security and compliance vulnerability assessment
- Practical recommendations for risk mitigation and architectural improvements
- Clear risk rating and go/no-go recommendation for the proposal

### Output Quality Standards
- **Technical Accuracy**: All technical analysis must be accurate and well-reasoned
- **Risk Realism**: Risk assessments based on realistic production scenarios
- **Constructive Tone**: Critical but professional, focused on improving outcomes
- **Actionable Insights**: Recommendations must be specific and implementable
- **Complete Coverage**: All aspects of the architecture thoroughly examined

---

*This prompt framework ensures comprehensive, expert-level critique of architectural proposals, identifying risks and failure modes that could impact production systems at scale. Use this framework to conduct thorough analysis that enables informed architectural decisions and risk mitigation strategies.*