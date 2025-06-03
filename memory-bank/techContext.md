# RAGKA v2 - Technical Implementation Context

## Technology Stack

### Backend Framework
- **Quart**: Async Python web framework (Flask-compatible)
- **Azure OpenAI**: GPT models with data source integration
- **Azure Search**: Vector and hybrid search capabilities
- **Pydantic**: Data validation and settings management

### Frontend
- **React**: JavaScript library for building user interfaces.
- **TypeScript**: Superset of JavaScript that adds static typing.
- **Vite**: Build tool for fast frontend development.
- **Fluent UI React**: Microsoft's React component library for UI.
- **CSS Modules**: For locally scoped CSS styles (`.module.css` files).

### Data Sources
- **Azure Cognitive Search**: Primary vector search
- **Elasticsearch**: Alternative search backend
- **Pinecone**: Vector database option
- **Azure CosmosDB**: Document storage with vector search
- **Azure ML Index**: Machine learning integrated search
- **Azure SQL Server**: Relational database integration
- **MongoDB**: Document database option

## Current Architecture Analysis

### Request Flow
1. **Frontend** (`frontend/src/pages/chat/Chat.tsx`) initiates API call to `/conversation`
2. **App Router** (`app.py`) → `conversation()` endpoint
3. **Request Processing** → `prepare_model_args()`
4. **Azure OpenAI** → Chat completion with data sources
5. **Response Formatting** → `format_non_streaming_response()` or `format_stream_response()`
6. **Frontend Display** → Parse response and show citations

### Key Files and Functions

#### `frontend/src/pages/chat/Chat.tsx` - Main Chat Page Component
- Orchestrates the chat interface, manages conversation state.
- Integrates `QuestionInput` and `Answer` components.
- Handles API calls to the backend.

#### `frontend/src/components/QuestionInput/QuestionInput.tsx` - Input Component
- Provides the text area for users to type questions.
- Implements dynamic height adjustment and rounded corners.
- Manages its own state for text and optional image uploads.
- Calls an `onSend` prop (passed from `Chat.tsx`) to submit the query.

#### `app.py` - Main Backend Application
```python
# Key functions:
# - prepare_model_args(request_body, request_headers) # Handles request setup
# - conversation() # Main endpoint for chat interactions
# Relevant functions for request handling and Azure OpenAI interaction.
```

#### `backend/utils.py` - Response Formatting
```python
# Key functions to enhance:
- format_non_streaming_response(chatCompletion, history_metadata, apim_request_id)
- format_stream_response(chatCompletionChunk, history_metadata, apim_request_id)
- format_pf_non_streaming_response(chatCompletion, history_metadata, response_field_name, citations_field_name)
```

#### `backend/settings.py` - Configuration
```python
# Key settings classes:
- _AzureOpenAISettings: Model configuration
- _SearchCommonSettings: Search parameters
- DatasourcePayloadConstructor: Data source integration
```

## Reference Implementation Analysis

### From `static/.ignore_context.py`

#### 1. Context Preparation Pattern
```python
def _prepare_context(self, results: List[Dict]) -> Tuple[str, Dict]:
    entries, src_map = [], {}
    sid = 1
    for res in results[:5]:  # TOP 5 SOURCES
        chunk = res["chunk"].strip()
        if not chunk:
            continue
        entries.append(f'<source id="{sid}">{chunk}</source>')
        src_map[str(sid)] = {
            "title": res["title"],
            "content": chunk
        }
        sid += 1
    return "\n\n".join(entries), src_map
```

#### 2. System Message Integration
```python
# Template with placeholders
system_prompt = """
### Task:
Respond to the user query using the provided context, incorporating inline citations in the format [id] **only when the <source> tag includes an explicit id attribute** (e.g., <source id="1">).

<context>
{{CONTEXT}}
</context>

<user_query>
{{QUERY}}
</user_query>
"""

# Message structure
messages = [
    {"role": "system", "content": processed_system_prompt},
    {"role": "user", "content": processed_user_content}
]
```

#### 3. Citation Filtering and Renumbering
```python
def _filter_cited(self, answer: str, src_map: Dict) -> List[Dict]:
    cited_sources = []
    for sid, sinfo in src_map.items():
        if f"[{sid}]" in answer:
            cited_sources.append({
                "id": sid,
                "title": sinfo["title"],
                "content": sinfo["content"],
                **({"url": sinfo["url"]} if "url" in sinfo else {})
            })
    return cited_sources

# Renumbering logic
renumber_map = {}
cited_sources = []
for new_id, src in enumerate(cited_raw, 1):
    old_id = src["id"]
    renumber_map[old_id] = str(new_id)
    entry = {"id": str(new_id), "title": src["title"], "content": src["content"]}
    if "url" in src:
        entry["url"] = src["url"]
    cited_sources.append(entry)

for old, new in renumber_map.items():
    answer = re.sub(rf"\[{old}\]", f"[{new}]", answer)
```

## Implementation Strategy

### Phase 1: Citation Processing Module

Create `backend/citation_processor.py`:

```python
import re
import json
import logging
from typing import List, Dict, Tuple, Optional

logger = logging.getLogger(__name__)

class CitationProcessor:
    """Handles source embedding, citation extraction, and renumbering"""
    
    @staticmethod
    def prepare_context_with_sources(sources: List[Dict]) -> Tuple[str, Dict]:
        """
        Convert RAG sources to <source id="n"> format
        Adapted from .ignore_context.py _prepare_context()
        """
        entries, src_map = [], {}
        sid = 1
        
        for source in sources[:5]:  # Top 5 sources
            # Extract content from various source formats
            content = ""
            if "chunk" in source:
                content = source["chunk"].strip()
            elif "content" in source:
                content = source["content"].strip()
            elif "text" in source:
                content = source["text"].strip()
            
            if not content:
                continue
                
            # Build source tag
            entries.append(f'<source id="{sid}">{content}</source>')
            
            # Build source mapping
            src_map[str(sid)] = {
                "title": source.get("title", source.get("document_id", "Untitled")),
                "content": content,
                "url": source.get("url", source.get("filepath", ""))
            }
            sid += 1
            
        return "\n\n".join(entries), src_map
    
    @staticmethod
    def filter_cited_sources(answer: str, src_map: Dict) -> List[Dict]:
        """
        Extract only sources actually cited in the response
        Adapted from .ignore_context.py _filter_cited()
        """
        cited_sources = []
        for sid, sinfo in src_map.items():
            if f"[{sid}]" in answer:
                source_entry = {
                    "id": sid,
                    "title": sinfo["title"],
                    "content": sinfo["content"]
                }
                if sinfo.get("url"):
                    source_entry["url"] = sinfo["url"]
                cited_sources.append(source_entry)
        return cited_sources
    
    @staticmethod
    def renumber_citations(answer: str, cited_sources: List[Dict]) -> Tuple[str, List[Dict]]:
        """
        Renumber citations sequentially based on appearance order
        """
        if not cited_sources:
            return answer, []
            
        # Create renumbering map
        renumber_map = {}
        renumbered_sources = []
        
        for new_id, src in enumerate(cited_sources, 1):
            old_id = src["id"]
            renumber_map[old_id] = str(new_id)
            
            # Create renumbered source entry
            entry = {
                "id": str(new_id),
                "title": src["title"],
                "content": src["content"]
            }
            if "url" in src:
                entry["url"] = src["url"]
            renumbered_sources.append(entry)
        
        # Apply renumbering to answer text
        renumbered_answer = answer
        for old, new in renumber_map.items():
            renumbered_answer = re.sub(rf"\[{old}\]", f"[{new}]", renumbered_answer)
        
        return renumbered_answer, renumbered_sources
    
    @classmethod
    def process_citations(cls, answer: str, sources: List[Dict]) -> Tuple[str, List[Dict]]:
        """
        Complete citation processing pipeline
        """
        try:
            # Prepare context and source mapping
            context, src_map = cls.prepare_context_with_sources(sources)
            
            # Filter to only cited sources
            cited_sources = cls.filter_cited_sources(answer, src_map)
            
            # Renumber citations sequentially
            final_answer, final_sources = cls.renumber_citations(answer, cited_sources)
            
            logger.info(f"Processed {len(sources)} sources, {len(final_sources)} cited")
            return final_answer, final_sources
            
        except Exception as e:
            logger.error(f"Citation processing error: {e}")
            return answer, []
```

### Phase 2: System Message Enhancement

Update system message in `backend/settings.py`:

```python
class _AzureOpenAISettings(BaseSettings):
    # ... existing fields ...
    
    system_message: str = """### Task:

Respond to the user query using the provided context, incorporating inline citations in the format [id] **only when the <source> tag includes an explicit id attribute** (e.g., <source id="1">).

### Guidelines:

- If you don't know the answer, clearly state that.
- If uncertain, ask the user for clarification.
- Respond in the same language as the user's query.
- If the context is unreadable or of poor quality, inform the user and provide the best possible answer.
- If the answer isn't present in the context but you possess the knowledge, explain this to the user and provide the answer using your own understanding.
- **Only include inline citations using [id] (e.g., [1], [2]) when the <source> tag includes an id attribute.**
- Do not cite if the <source> tag does not contain an id attribute.
- Do not use XML tags in your response.
- Ensure citations are concise and directly related to the information provided.

### Example of Citation:

If the user asks about a specific topic and the information is found in a source with a provided id attribute, the response should include the citation like in the following example:

* "According to the study, the proposed method increases efficiency by 20% [1]."

### Output:

Provide a clear and direct response to the user's query, including inline citations in the format [id] only when the <source> tag with id attribute is present in the context."""
```

### Phase 3: Integration Points

#### A. Context Injection in `app.py`
```python
def prepare_model_args(request_body, request_headers):
    # ... existing code ...
    
    # NEW: Handle context preparation for citation system
    if app_settings.datasource and len(messages) > 0 and messages[-1]["role"] == "user":
        user_query = messages[-1]["content"]
        
        # The system message will be enhanced to include context injection
        # This will be handled in the response processing phase
        pass
    
    # ... rest of existing code ...
```

#### B. Response Processing Enhancement
```python
# In backend/utils.py
from backend.citation_processor import CitationProcessor

def format_non_streaming_response(chatCompletion, history_metadata, apim_request_id):
    # ... existing code ...
    
    if len(chatCompletion.choices) > 0:
        message = chatCompletion.choices[0].message
        if message:
            # Handle context and citations
            sources = []
            if hasattr(message, "context"):
                context_data = message.context
                if isinstance(context_data, str):
                    context_data = json.loads(context_data)
                sources = context_data.get("citations", [])
            
            # Process citations if sources are available
            if sources:
                processed_answer, cited_sources = CitationProcessor.process_citations(
                    message.content, sources
                )
                
                response_obj["choices"][0]["messages"] = [
                    {"role": "assistant", "content": processed_answer},
                    {"role": "tool", "content": json.dumps({"sources": cited_sources})}
                ]
            else:
                # Fallback to original behavior
                response_obj["choices"][0]["messages"].append({
                    "role": "assistant",
                    "content": message.content,
                })
```

## Testing Strategy

### Unit Tests
- Citation extraction accuracy
- Renumbering logic validation
- Source filtering correctness
- URL field preservation

### Integration Tests
- End-to-end citation flow
- Multiple data source compatibility
- Streaming vs non-streaming responses
- Error handling and fallbacks

### Performance Considerations
- Context window optimization
- Source content truncation
- Efficient regex operations
- Memory usage with large source sets

## Deployment Considerations

### Backward Compatibility
- Graceful fallback when no sources available
- Existing API response structure preserved
- Frontend handles both old and new citation formats

### Configuration
- Citation system can be enabled/disabled
- Maximum source count configurable
- Source content length limits
- URL field inclusion optional

### Monitoring
- Citation accuracy metrics
- Source utilization tracking
- Performance impact measurement
- Error rate monitoring

This technical context provides the foundation for implementing the citation enhancement system while maintaining the robustness and scalability of the existing RAGKA v2 architecture.
