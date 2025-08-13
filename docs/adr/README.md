# Architecture Decision Records Index

This directory contains all Architecture Decision Records (ADRs) for the AI Resume Builder project. ADRs document important architectural decisions made during the development process.

## ADR Index

| ADR | Title | Date | Status |
|-----|-------|------|--------|
| [001](./001-tech-stack-selection.md) | Technology Stack Selection | 2025-08-13 | Accepted |
| [002](./002-phase1-workflow-architecture.md) | Phase 1 Workflow Architecture | 2025-08-13 | Accepted |
| [003](./003-n8n-integration-architecture.md) | n8n Integration Architecture | 2025-08-13 | Accepted |
| [004](./004-data-storage-security-architecture.md) | Data Storage and Security Architecture | 2025-08-13 | Accepted |
| [005](./005-frontend-architecture-ux.md) | Frontend Architecture and User Experience | 2025-08-13 | Accepted |

## ADR Status Definitions

- **Proposed**: Under review and discussion
- **Accepted**: Approved and being implemented
- **Superseded**: Replaced by a newer decision
- **Deprecated**: No longer recommended but still in use
- **Rejected**: Considered but not adopted

## Quick Architecture Overview

### Technology Stack
- **Frontend**: React 18 + TypeScript + Vite + Tailwind CSS + shadcn/ui
- **Backend**: Firebase (Auth, Firestore, Storage, Functions)
- **AI Processing**: n8n workflows + OpenAI GPT-4
- **Build Tools**: Vite for frontend, Firebase CLI for backend

### Core Architecture Patterns
- **Frontend**: Component-based React architecture with context state management
- **Backend**: Serverless Firebase Functions with Firestore NoSQL database
- **AI Integration**: Event-driven n8n workflows triggered by database changes
- **Security**: Multi-layered security with Firebase rules and encryption

### Phase 1 Workflow
1. User uploads CV (stored in Firebase Storage + parsed content in Firestore)
2. User submits job descriptions (stored in Firestore)
3. User requests resume enhancement (triggers n8n workflow via Cloud Function)
4. AI processes and enhances resume content (OpenAI API)
5. Enhanced resume stored and available for download (PDF generation)

### Data Flow
```
React UI → Firebase Auth → Firestore → Cloud Functions → n8n → OpenAI → Firebase → Real-time UI Updates
```

## Contributing to ADRs

When making significant architectural decisions:

1. Create a new ADR using the next sequential number
2. Follow the established template structure
3. Include context, decision, rationale, and consequences
4. Update this index file
5. Get team review before marking as "Accepted"

## Template Structure

Each ADR should include:
- **Date**: When the decision was made
- **Status**: Current status of the decision
- **Context**: The situation that led to the decision
- **Decision**: What was decided
- **Rationale**: Why this decision was made
- **Consequences**: Expected positive and negative outcomes
- **Alternatives Considered**: Other options that were evaluated
- **Implementation Notes**: Key implementation details
- **Review Date**: When to reassess the decision

## Review Schedule

All ADRs should be reviewed:
- After major milestones (MVP completion, 1000 users, etc.)
- When technology landscape changes significantly
- If implementation reveals major issues with the decisions
- At least annually for foundational decisions

For questions about any architectural decision, refer to the specific ADR or contact the development team.
