# Neel's Commenting Style Implementation Demo

This document demonstrates how Neel's commenting style would be applied to the neo-localmcp codebase.

## Core Principles Applied

1. **Concise Breadcrumbs**: Using `->` notation to show data flow
2. **Concept Tags**: Using `#` tags for quick categorization
3. **Stream-of-Thought Friendly**: Comments that complement code reading, not repeat it
4. **Focused on Intent and Flow**: Not syntax explanations

## Sample Implementation

### Before (Traditional Style)
```python
def prepare_context(task: str, repo_root: str = "auto", token_budget: int = 3000, max_files: int = 6, use_ollama: bool = False, model: str | None = None, output_format: str = "mcp_text", *, record: bool = True) -> str:
    """MCP/CLI-facing adapter -- see context_prepare for the core implementation."""
    return context_prepare(task, repo_root, max_files=None, limit=max_files, use_ollama=use_ollama, model=model, output_format=output_format, token_budget=token_budget, record=record)
```

### After (Neel's Style)
```python
def prepare_context(task: str, repo_root: str = "auto", token_budget: int = 3000, max_files: int = 6, use_ollama: bool = False, model: str | None = None, output_format: str = "mcp_text", *, record: bool = True) -> str:
    """MCP/CLI-facing adapter -- see context_prepare for the core implementation."""
    # task -> context_prepare -> response
    return context_prepare(task, repo_root, max_files=None, limit=max_files, use_ollama=use_ollama, model=model, output_format=output_format, token_budget=token_budget, record=record)  

# #StateFlow
def context_prepare(task: str, repo_root: str = "auto", max_files: int | None = None, limit: int = 6, use_ollama: bool = False, model: str | None = None, output_format: str = "json", token_budget: int = 3000, *, record: bool = True) -> str:
    """
    Core retrieval implementation; prepare_context is the MCP/CLI adapter over this.
    #StateFlow
    task -> normalize_query -> intent classification -> ranking -> candidate selection -> excerpts
    """
    # #Binding
    root = repo_root_or_cwd(repo_root) 
    
    # repo_status -> repo_memory.status -> context_hash
    status = repo_memory.status(root)
    current_hash = repo_memory.context_hash(root)
    
    # #Timing
    begin = time.perf_counter()
    
    # #Ranking
    interpreted = normalize_query(task)
    intent = interpreted.get("intent")
    
    # #InputValidation
    if not interpreted.get("query"):
        return json_out({"ok": False, "error": "empty query"})
        
    # #Cache
    cached = repo_memory.get_cached_context(current_hash)
    if cached and isinstance(cached, dict) and cached.get("response"):
        # #CacheHit
        resp = cached.get("response")
        if record:
            repo_memory.record_task_query(task, current_hash, intent, resp)  # #Persistence
        return json_out(resp)
        
    # #Processing
    # task -> interpretation -> category boosting -> symbol search -> scoring -> ranking
    terms = interpreted.get("terms") or []
    strong = [t for t in terms if t in interpreted.get("strong_terms", [])] 
    weak = [t for t in terms if t in interpreted.get("weak_terms", [])]
    
    # #SymbolHits  
    symbol_hits = repo_memory.symbol_search(root, strong, weak, intent)
    
    # #FileScoring
    candidates, symbol_hits = _score_index_and_symbol_hits(terms, interpreted, root, limit, intent)
    
    # #OllamaRanking
    if use_ollama and model:
        # #Ranking
        ranking = _run_ollama_ranking(task, interpreted, candidates, limit, model)
        if ranking and ranking.get("ok"):
            candidates = ranking.get("ranked_candidates", [])
            
    # #FileSelection
    read_first, others = _select_read_first(candidates, set(), intent, limit, max_files)
    
    # #ContentExtraction
    excerpts = repo_memory.file_excerpts(
        _build_excerpt_ranges(read_first, symbol_hits, interpreted)[0],
        root,
        max_chars=token_budget  # #MemoryManagement
    )
    
    # response -> formatting -> output
    resp = {
        # #ResponseStructure
        "ok": True,
        "interpreted_query": interpreted,
        "repo_status": status,
        "read_first": read_first,
        "candidate_files": others,
        "context_excerpts": excerpts,
        "token_budget": token_budget,
        "agent_guidance": _agent_guidance(read_first, interpreted),
        "retrieval_metrics": {"estimated_tokens_returned": token_budget},
        "ollama_requested": use_ollama,
        "mode": "agent_ready_natural_context",
    }
    
    # #Finalization
    formatted = _format(resp, output_format)
    
    # #Persistence
    if record and not use_ollama:
        repo_memory.record_task_query(task, current_hash, intent, resp)
        
    return formatted
```

## Benefits of This Approach

1. **Navigation**: Developers can quickly understand data movement through the system
2. **Quick Categorization**: Concept tags like `#StateFlow` and `#Binding` help mentally organize code sections
3. **Reduced Cognitive Load**: No need to read through verbose syntactic explanations
4. **Improved Collaboration**: Clearer communication about complex workflows

## Enforcement Strategy

To enforce this in the repository:
1. Add this style guide as documentation
2. Include in code review guidelines
3. Consider adding to linter rules for consistency
4. Train team members on the approach