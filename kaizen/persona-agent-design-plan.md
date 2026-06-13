# Plan: Persona-Based Agent Design Utility

## Goal
Design a reusable TypeScript/React utility that programmatically creates customized LLM agent personas and skills based on user preferences, allowing applications to mimic specific personalities and expertise on demand.

## Current Context & Assumptions
- **Tech Stack**: TypeScript, React.
- **LLM Integration**: OpenRouter (API keys and model selection injected by the host application).
- **Output**: A structured set of prompts (Main Persona Prompt + Specialized Skill Prompts).
- **Vault Location**: Documentation and design notes reside in `/home/todd/e7/social/agents`.
- **Scope**: Utility logic only; no UI implementation.

## Proposed Approach
The utility will be implemented as a set of services/hooks that manage a state machine for the "Agent Design Pipeline." It will separate the *gathering* of preferences, the *research* of the persona, and the *generation* of the final configuration.

### Pipeline Architecture
1. **Preference Capture**: Interface to collect `Genre`, `Personalities`, and `User Details`.
2. **Persona Synthesis (Research Phase)**: 
   - Use an LLM to research the selected personalities and synthesize their core traits.
   - Combine research with user-provided "why I like them" details.
3. **Artifact Generation**:
   - **Main Prompt**: A high-level identity prompt defining traits, voice, and characteristics.
   - **Skill Library**: A map of specific capabilities (e.g., "Signature Cooking") that can be injected into the context only when needed.

## Step-by-Step Plan

### Phase 1: Core Type Definitions & Interfaces
- Define `PersonaPreferences` (Genre, Personality list, User notes).
- Define `AgentConfiguration` (Main prompt, Skill map).
- Define `LLMProvider` interface to allow the calling app to inject OpenRouter config.

### Phase 2: Research Service Implementation
- Create a `PersonaResearchService` that:
  - Takes preferences as input.
  - Formulates research queries for the LLM to extract specific traits of the mentioned personalities.
  - Aggregates findings into a "Persona Blueprint."

### Phase 3: Generation Logic
- Create an `AgentGeneratorService` that:
  - Converts the "Persona Blueprint" into a structured `Main Prompt`.
  - Identifies potential "Skills" based on the persona's strengths/genre and generates separate prompts for them.
  - Ensures the output format is compatible with standard LLM system message patterns.

### Phase 4: React Integration Layer
- Implement a custom hook (e.g., `useAgentDesigner`) to manage the multi-step state.
- Provide methods to transition between steps: `startDesign()`, `submitPreferences()`, `generateAgent()`.

## Files Likely to be Created/Changed
- `src/utilities/agent-designer/types.ts` - Core interfaces.
- `src/utilities/agent-designer/research-service.ts` - Logic for persona synthesis.
- `src/utilities/agent-designer/generator-service.ts` - Logic for prompt/skill creation.
- `src/utilities/agent-designer/use-agent-designer.ts` - React hook for state management.
- `src/utilities/agent-designer/llm-client.ts` - Wrapper for OpenRouter calls.

## Tests & Validation
- **Unit Tests**: Validate that the `GeneratorService` produces prompts containing key traits from the `ResearchService`.
- **Integration Tests**: Mock OpenRouter responses to ensure the full pipeline (Preferences -> Research -> Prompts) completes without state loss.
- **Schema Validation**: Ensure the generated `AgentConfiguration` matches the expected format for the calling application.

## Risks, Tradeoffs, and Open Questions
- **Risk**: LLM "hallucinations" during the research phase if personalities are obscure.
- **Tradeoff**: Deciding between a single large prompt vs. many small skills. *Decision: Use the "Main Prompt + Skills" model to minimize token usage and maintain precision.*
- **Open Question**: How strictly should the "Skills" be categorized? (e.g., should the utility suggest skills or should the user define them during the design phase?)
