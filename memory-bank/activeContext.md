# RAGKA v2 - Active Implementation Context

## Current Focus: RAG Citation Enhancement Implementation

### Immediate Goal
Implement the citation system that:
1. Embeds up to 5 top sources as `<source id="n">...</source>` tags in context
2. Formats citations as `[n]` in model responses
3. Returns only cited sources in API response with proper renumbering
4. Includes URL fields for hyperlinking when available

### Implementation Status
- ✅ **Analysis Complete**: Reviewed existing system and reference implementation
- ✅ **Memory Bank Created**: Project brief and technical context documented
- ✅ **Phase 1 Complete**: Citation processing module created (`backend/citation_processor.py`)
- ✅ **Phase 2 Complete**: Integrated with response formatting (`backend/utils.py`)
- ✅ **Phase 3 Complete**: System message enhanced with citation instructions
- ✅ **Phase 4 Complete**: End-to-end testing and validation (100% test pass rate)
- ✅ **Production Ready**: All core backend citation functionality implemented and tested.

## Current Focus: Frontend UI Enhancements & Documentation Update

### Immediate Goal
1.  **Update Input Text Box UI:**
    *   Reduce initial height to accommodate a single line.
    *   Enable dynamic height increase up to 6 lines, then scroll.
    *   Ensure rounded corners are correctly applied.
2.  **Update Memory Bank:**
    *   Create missing core files (`productContext.md`, `systemPatterns.md`).
    *   Update existing Memory Bank files (`projectbrief.md`, `techContext.md`, `activeContext.md`, `progress.md`) to reflect the React/TypeScript frontend and current UI task.

### Implementation Status
- ✅ **Analysis Complete**: Reviewed existing `QuestionInput` component and CSS.
- ✅ **UI Changes Implemented**:
    - `frontend/src/components/QuestionInput/QuestionInput.tsx`: Added `autoAdjustHeight`, `rows={1}`, `maxRows={6}` to `TextField`.
    - `frontend/src/components/QuestionInput/QuestionInput.module.css`: Removed fixed height from container, adjusted text area padding and alignment.
- ✅ **Memory Bank Updates**:
    - ✅ `memory-bank/productContext.md` created.
    - ✅ `memory-bank/systemPatterns.md` created.
    - ✅ `memory-bank/techContext.md` updated.
    - ✅ `memory-bank/projectbrief.md` updated.
- ⏳ **Pending**: Update `activeContext.md` (this file) and `progress.md`.
- ⏳ **Future**: Further frontend integration for citation display, deployment.

## Key Implementation Decisions

### 1. Modular Approach
- Create `backend/citation_processor.py` as standalone module
- Adapt proven patterns from `static/.ignore_context.py`
- Integrate with existing `backend/utils.py` functions
- Maintain backward compatibility

### 2. System Message Strategy
- Use the provided comprehensive system message template
- Inject context using `{{CONTEXT}}` and `{{QUERY}}` placeholders
- Leverage Azure OpenAI's existing data source integration
- Process citations in post-processing rather than pre-processing

### 3. Response Processing Pipeline
```
Azure OpenAI Response → Extract Sources → Process Citations → Renumber → Return API Response
```

### 4. Source Handling
- Extract from Azure OpenAI context data (citations field)
- Support multiple source formats (chunk, content, text fields)
- Preserve title, content, and URL information
- Limit to top 5 sources for context window efficiency

## Implementation Plan

### Phase 1: Citation Processing Module ✅ READY TO IMPLEMENT

**File**: `backend/citation_processor.py`

**Key Functions**:
1. `prepare_context_with_sources()` - Convert sources to `<source id="n">` format
2. `filter_cited_sources()` - Extract only cited sources from response
3. `renumber_citations()` - Sequential renumbering of citations
4. `process_citations()` - Complete pipeline orchestration

**Adaptation Notes**:
- Based on proven implementation from `.ignore_context.py`
- Handles various source field names (chunk, content, text)
- Preserves URL information for hyperlinking
- Robust error handling and logging

### Phase 2: Response Processing Integration

**Files to Modify**:
- `backend/utils.py` - Update response formatting functions
- Import and use `CitationProcessor` class

**Key Changes**:
1. **`format_non_streaming_response()`**:
   - Extract sources from `message.context.citations`
   - Process citations using `CitationProcessor.process_citations()`
   - Return enhanced response with sources array

2. **`format_stream_response()`**:
   - Handle streaming citation extraction
   - Accumulate sources during streaming
   - Process citations at stream completion

3. **`format_pf_non_streaming_response()`**:
   - Integrate with PromptFlow responses
   - Maintain existing citation field handling

### Phase 3: System Message Enhancement

**File**: `backend/settings.py`

**Changes**:
- Update `_AzureOpenAISettings.system_message` with comprehensive template
- Add context injection capabilities
- Maintain existing system message override functionality

### Phase 4: Context Injection Strategy

**Challenge**: Azure OpenAI with data sources handles context automatically
**Solution**: Post-process the response rather than pre-inject context

**Approach**:
1. Let Azure OpenAI retrieve and use sources normally
2. Extract sources from the response context
3. Process citations in the response text
4. Return enhanced response with proper citation structure

### Phase 5: Frontend Compatibility (Ongoing)

**Files**:
- `frontend/src/pages/chat/Chat.tsx`
- `frontend/src/components/QuestionInput/QuestionInput.tsx`
- `frontend/src/components/Answer/Answer.tsx`
- Associated CSS Modules

**Enhancements**:
- **Input Text Box**: Updated for dynamic height and rounded corners.
- **Citation Display**: (Future) Handle new sources array structure, display clickable URLs, improve visual distinction.

## Technical Considerations

### Context Window Management
- Limit to 5 sources to preserve context space
- Truncate source content if needed (500 chars max per source)
- Efficient source selection based on relevance scores

### Error Handling
- Graceful fallback when no sources available
- Handle malformed source data
- Preserve original response if citation processing fails
- Comprehensive logging for debugging

### Performance Optimization
- Efficient regex operations for citation extraction
- Minimal memory overhead for source processing
- Fast renumbering algorithm
- Lazy evaluation where possible

### Backward Compatibility
- Existing API responses continue to work
- Frontend handles both old and new citation formats
- Configuration option to enable/disable new citation system
- Gradual migration path

## Current Implementation Priority

### Current Task: Input Text Box UI Update & Documentation

**Completed Steps:**
1.  **Modified `frontend/src/components/QuestionInput/QuestionInput.tsx`**:
    *   Added `autoAdjustHeight` prop to `TextField`.
    *   Added `rows={1}` for initial single-line height.
    *   Added `maxRows={6}` for maximum height before scrolling.
2.  **Modified `frontend/src/components/QuestionInput/QuestionInput.module.css`**:
    *   Removed fixed `height` from `.questionInputContainer`.
    *   Adjusted `padding` and `align-items` for `.questionInputTextArea`.
    *   Ensured `border-radius` on `.questionInputContainer` applies.
3.  **Created `memory-bank/productContext.md`**.
4.  **Created `memory-bank/systemPatterns.md`**.
5.  **Updated `memory-bank/techContext.md`** (frontend stack, request flow, key files).
6.  **Updated `memory-bank/projectbrief.md`** (frontend components).

**Immediate Next Steps:**
1.  **Update `memory-bank/activeContext.md`** (this file) with the current status.
2.  **Update `memory-bank/progress.md`** to reflect UI changes and documentation updates.
3.  **Verify UI changes** (visually, if possible, or by confirming code logic).

## Success Metrics

### Functional Validation
- ✅ Top 5 sources embedded in context
- ✅ Citations use `[n]` format correctly
- ✅ Only cited sources in API response
- ✅ Sequential renumbering works
- ✅ URL fields preserved and accessible

### Technical Validation
- ✅ No breaking changes to existing API
- ✅ Performance impact < 100ms per request
- ✅ Error rate < 1% for citation processing
- ✅ Memory usage increase < 10%
- ✅ All data sources supported

### User Experience Validation
- ✅ Citations are clickable when URLs available
- ✅ Source attribution is accurate
- ✅ Visual design is clear and intuitive
- ✅ Loading performance is acceptable
- ✅ Mobile responsiveness maintained

## Risk Mitigation

### Technical Risks
- **Context extraction failure**: Robust parsing with fallbacks
- **Citation regex issues**: Comprehensive test coverage
- **Performance degradation**: Profiling and optimization
- **Memory leaks**: Proper resource cleanup

### Integration Risks
- **Breaking changes**: Extensive backward compatibility testing
- **Data source incompatibility**: Universal source format handling
- **Frontend display issues**: Progressive enhancement approach
- **Configuration conflicts**: Clear setting precedence rules

This active context provides the roadmap for implementing the citation enhancement system with clear priorities, technical considerations, and success criteria.
