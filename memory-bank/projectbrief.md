# RAGKA v2 - RAG Citation Enhancement Project Brief

## Project Overview

RAGKA v2 is a Retrieval Augmented Generation Knowledge Assistant built on Azure OpenAI and Azure Search. The current implementation needs enhancement to provide proper citation handling with embedded sources, numbered citations, and accurate source tracking.

## Core Requirements

### 1. Source Embedding in Context
- Include up to 5 top sources embedded as `<source id="n">...</source>` tags in the context sent to the model
- Sources should be extracted from the RAG retrieval results
- Each source should have a unique sequential ID (1, 2, 3, 4, 5)

### 2. Citation Formatting in Model Response
- Model should format citations using `[n]` format (e.g., [1], [2]) corresponding to each source
- Citations should only reference sources that are actually embedded in the context
- Model should be instructed to only cite when `<source id="n">` tags are present

### 3. Response API Structure
- Return a `sources` array in the API response with correct id, title, and content
- Only include sources that are actually cited (appear as [n] in the answer)
- Renumber citations in the final answer to match the order they appear
- Include URL field for each source when available for UI hyperlinking

### 4. Citation Renumbering Logic
- If source with id="4" appears first in the response, it becomes [1]
- Sequential renumbering based on order of appearance in the response
- Update both the response text and the sources array accordingly

## Current System Architecture

### Backend Components
- **app.py**: Main Quart application with conversation endpoints
- **backend/utils.py**: Response formatting utilities
- **backend/settings.py**: Configuration management for data sources

### Frontend Components
- **React/TypeScript**: Main application built with React and TypeScript.
- **Vite**: Build tool and development server.
- **Fluent UI**: Component library for UI elements.
- **`frontend/src/pages/chat/Chat.tsx`**: Main chat page component.
- **`frontend/src/components/QuestionInput/QuestionInput.tsx`**: Component for user text input.

### Data Sources Supported
- Azure Cognitive Search
- Elasticsearch  
- Pinecone
- Azure CosmosDB Mongo vCore
- Azure ML Index
- Azure SQL Server
- MongoDB

### Current Citation Handling
- Basic citation support exists in `format_pf_non_streaming_response()`
- Citations passed through as tool messages with JSON content
- Frontend displays citations in dedicated section
- Missing: embedded sources, proper numbering, filtering

## Reference Implementation

The file `static/.ignore_context.py` contains a complete working implementation of the desired citation system with:

1. **Context Preparation** (`_prepare_context()`)
   - Embeds top 5 sources as `<source id="n">content</source>`
   - Creates source mapping for later reference
   - Handles empty/invalid sources gracefully

2. **Citation Filtering** (`_filter_cited()`)
   - Extracts only sources actually cited in the response
   - Preserves title, content, and URL information
   - Uses regex to find `[n]` patterns in response

3. **Renumbering Logic**
   - Sequential renumbering of citations (1, 2, 3...)
   - Updates both response text and sources array
   - Maintains consistency between citations and source list

4. **System Message Template**
   - Comprehensive instructions for citation behavior
   - Explicit rules about when to cite sources
   - Template structure with `{{CONTEXT}}` and `{{QUERY}}` placeholders

## Success Criteria

1. **Functional Requirements**
   - ✅ Top 5 sources embedded in context with `<source id="n">` tags
   - ✅ Model responses use `[n]` citation format
   - ✅ Only cited sources returned in API response
   - ✅ Citations renumbered based on appearance order
   - ✅ URL fields included when available

2. **Technical Requirements**
   - ✅ Works with existing Azure OpenAI integration
   - ✅ Compatible with all supported data sources
   - ✅ Handles both streaming and non-streaming responses
   - ✅ Maintains backward compatibility with existing API

3. **User Experience**
   - ✅ Clickable citations with URLs when available
   - ✅ Clear visual distinction of cited sources
   - ✅ Accurate source attribution
   - ✅ Responsive citation display

## Implementation Strategy

The implementation will adapt the proven patterns from `static/.ignore_context.py` to work within the existing RAGKA v2 architecture, focusing on:

1. **Minimal Disruption**: Enhance existing functions rather than replacing them
2. **Modular Design**: Create reusable citation processing components
3. **Comprehensive Testing**: Ensure compatibility across all data sources
4. **Performance Optimization**: Efficient source processing and citation extraction

This approach leverages the working implementation while maintaining the robustness and scalability of the current system.\
