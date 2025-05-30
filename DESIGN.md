# Design Document: ElixirScope.TemporalDebug (elixir_scope_temporal_debug)

## 1. Purpose & Vision

**Summary:** Enables AST-aware time-travel debugging capabilities, including state reconstruction at arbitrary points in time and structural replay of execution, by enhancing basic temporal storage with CPG context.

**(Greatly Expanded Purpose based on your existing knowledge of ElixirScope and CPG features):**

The `elixir_scope_temporal_debug` library aims to provide a revolutionary "Execution Cinema" experience by allowing developers and AI assistants to navigate and inspect the execution history of an Elixir application not just chronologically, but also structurally through the lens of the Code Property Graph (CPG). It builds upon foundational event storage by deeply integrating AST/CPG context into the time-travel debugging process.

This library aims to:
*   **Provide AST-Aware State Reconstruction:** Allow reconstruction of a process's (or system's) state at any given past timestamp, and critically, augment this state with the precise CPG context (e.g., which CPG node was being executed, what was the CPG-level call stack) active at that moment. This is achieved by `TemporalBridgeEnhancement`.
*   **Enable Semantic Time-Travel:** Offer capabilities to navigate execution history based on CPG structural elements. For example, "show me all states where execution passed through CPG node X" or "jump to the previous/next execution of the function represented by CPG node Y."
*   **Facilitate Code-Centric Debugging:** Allow users to explore past executions by focusing on the code's structure (via the CPG) rather than just a linear timeline of events. This means understanding not just *what* state a variable had, but *where in the CPG* that state was relevant.
*   **Support Structural Replay:** Generate execution traces that depict the sequence of CPG nodes traversed over a time range, providing a high-level structural understanding of a past execution flow.
*   **Integrate with Core Temporal Storage:** Leverage an underlying `TemporalStorage` mechanism (likely from `elixir_scope_storage` or a dedicated part of this library) for efficient storage and retrieval of time-ordered events.
*   **Cache Enhanced Temporal Data:** Implement caching (`CacheManager` within `TemporalBridgeEnhancement`) for frequently requested reconstructed states or AST-aware traces to improve performance.

The core innovation of this library lies in the `TemporalBridgeEnhancement` component. It takes raw temporal event data and, by collaborating with `elixir_scope_correlator` and `elixir_scope_ast_repo`, enriches it with deep CPG context. This allows for a much richer and more intuitive debugging experience than traditional time-travel debuggers that only show event sequences or raw state.

This library will enable:
*   Developers to "rewind" and "replay" execution, seeing not just variable values but also the active CPG location and context at each step.
*   AI assistants interacting via `TidewaveScope` to answer complex temporal-structural questions, like "What was the CPG execution path that led to this variable having an incorrect value at timestamp T?"
*   A powerful "Execution Cinema" UI to visually represent past executions overlaid on the CPG or source code.
*   Faster root cause analysis by quickly navigating to relevant past states based on CPG structural queries.

## 2. Key Responsibilities

This library is responsible for:

*   **Temporal Event Storage (`TemporalStorage` - could be part of `elixir_scope_storage` or specific to this lib):**
    *   Storing runtime events in a way that is optimized for temporal queries (e.g., ordered by timestamp).
    *   Indexing events by session ID, timestamp, and potentially `ast_node_id` if it's the primary query axis for some features.
*   **Temporal Bridging (`TemporalBridge` - the original concept):**
    *   Providing an interface for `elixir_scope_capture_core` to send events for temporal storage.
    *   Handling basic state reconstruction based purely on event replay (if this functionality is preserved separately).
*   **AST-Enhanced Temporal Bridging (`TemporalBridgeEnhancement` GenServer):**
    *   Orchestrating the enhancement of temporal debugging with AST/CPG context.
    *   Interfacing with `elixir_scope_correlator` to get `ASTContext` for events and states.
    *   Interfacing with `elixir_scope_ast_repo` if further CPG details are needed beyond what the correlator provides.
    *   Managing its own cache (`CacheManager`) for reconstructed AST-enhanced states and traces.
*   **AST-Aware State Reconstruction (`StateManager` within `TemporalBridgeEnhancement`):**
    *   Taking a timestamp and session ID, retrieving relevant events from `TemporalStorage`.
    *   Replaying events to reconstruct the application/process state.
    *   For each reconstructed state or key event, using `ASTContextBuilder` to fetch and attach the CPG context.
*   **AST-Aware Trace Building (`TraceBuilder` within `TemporalBridgeEnhancement`):**
    *   Generating execution traces that show the sequence of CPG nodes executed over a time range.
    *   Identifying structural patterns or anomalies in these CPG-aware traces.
*   **Event Processing for Temporal Context (`EventProcessor` within `TemporalBridgeEnhancement`):**
    *   Querying and filtering events from `TemporalStorage` based on time, `ast_node_id`, or correlation ID for temporal analysis.

## 3. Key Modules & Structure

The primary modules within this library will be:

*   `ElixirScope.TemporalDebug.TemporalStorage` (GenServer/ETS manager for time-ordered events; could be an API wrapper around `elixir_scope_storage` with specific indexing)
*   `ElixirScope.TemporalDebug.TemporalBridge` (Optional: Original GenServer for basic temporal functions if kept distinct)
*   `ElixirScope.TemporalDebug.TemporalBridgeEnhancement` (Main GenServer for AST-aware features)
    *   `ElixirScope.TemporalDebug.TemporalBridgeEnhancement.StateManager`
    *   `ElixirScope.TemporalDebug.TemporalBridgeEnhancement.TraceBuilder`
    *   `ElixirScope.TemporalDebug.TemporalBridgeEnhancement.EventProcessor`
    *   `ElixirScope.TemporalDebug.TemporalBridgeEnhancement.ASTContextBuilder`
    *   `ElixirScope.TemporalDebug.TemporalBridgeEnhancement.CacheManager`
    *   `ElixirScope.TemporalDebug.TemporalBridgeEnhancement.Types`

### Proposed File Tree:

```
elixir_scope_temporal_debug/
├── lib/
│   └── elixir_scope/
│       └── temporal_debug/ # Or 'capture/temporal_bridge...' if following original
│           ├── temporal_storage.ex             # Manages time-ordered event store
│           ├── temporal_bridge.ex              # (Optional) Basic bridge
│           ├── temporal_bridge_enhancement.ex  # Main GenServer for enhanced features
│           └── temporal_bridge_enhancement/
│               ├── state_manager.ex
│               ├── trace_builder.ex
│               ├── event_processor.ex
│               ├── ast_context_builder.ex
│               ├── cache_manager.ex
│               └── types.ex
├── mix.exs
├── README.md
├── DESIGN.MD
└── test/
    ├── test_helper.exs
    └── elixir_scope/
        └── temporal_debug/
            ├── temporal_storage_test.exs
            └── temporal_bridge_enhancement_test.exs
```

**(Greatly Expanded - Module Description):**
*   **`ElixirScope.TemporalDebug.TemporalStorage`**: This component is responsible for storing events in a manner optimized for time-based queries. It will likely use ETS with `ordered_set` tables keyed by timestamp. It needs to efficiently retrieve event sequences for specific time ranges or leading up to a particular point. It may also maintain secondary indexes for `ast_node_id` or `correlation_id` if direct temporal queries on these are common.
*   **`ElixirScope.TemporalDebug.TemporalBridgeEnhancement` (GenServer):** This is the primary public interface for AST-aware time-travel features. It coordinates the different sub-modules.
    *   **`StateManager`**: Handles requests for state reconstruction. It queries `TemporalStorage` for relevant events, replays them to rebuild state, and then uses `ASTContextBuilder` to enrich that state with CPG information.
    *   **`TraceBuilder`**: Builds execution traces that highlight the flow through CPG nodes. It uses `EventProcessor` to get events and `ASTContextBuilder` (via `elixir_scope_correlator`) to map these events to CPG nodes.
    *   **`EventProcessor`**: Provides utilities to query and filter events from `TemporalStorage` specifically for temporal analysis needs (e.g., getting events associated with an AST node within a time window).
    *   **`ASTContextBuilder`**: (This module might be a thin wrapper around calls to `elixir_scope_correlator`.) Its role is to take a raw event or a reconstructed state and obtain the relevant `ASTContext` (which includes CPG node details) by interacting with `elixir_scope_correlator`.
    *   **`CacheManager`**: Manages ETS caches for frequently reconstructed AST-enhanced states or common AST-aware trace segments to speed up repeated queries.
    *   **`Types`**: Defines the core data structures specific to this library, such as `ast_enhanced_state()` and `ast_execution_trace()`.

## 4. Public API (Conceptual)

Via `ElixirScope.TemporalDebug.TemporalBridgeEnhancement` (GenServer):

*   `start_link(opts :: keyword()) :: GenServer.on_start()`
    *   Options: `temporal_storage_ref`, `ast_repo_ref`, `correlator_ref`.
*   `reconstruct_state_with_ast(session_id :: String.t(), timestamp :: non_neg_integer()) :: {:ok, ElixirScope.TemporalDebug.Types.ast_enhanced_state()} | {:error, term()}`
*   `get_ast_execution_trace(session_id :: String.t(), start_time :: non_neg_integer(), end_time :: non_neg_integer()) :: {:ok, ElixirScope.TemporalDebug.Types.ast_execution_trace()} | {:error, term()}`
*   `get_states_for_ast_node(session_id :: String.t(), ast_node_id :: String.t()) :: {:ok, list(ElixirScope.TemporalDebug.Types.ast_enhanced_state())} | {:error, term()}`
*   `get_execution_flow_between_cpg_nodes(session_id :: String.t(), from_cpg_node_id :: String.t(), to_cpg_node_id :: String.t(), time_range :: {integer(), integer()} | nil) :: {:ok, map_representing_flow} | {:error, term()}`
*   `get_temporal_stats() :: {:ok, map()}`
*   `clear_temporal_caches() :: :ok`

Via `ElixirScope.TemporalDebug.TemporalBridge` (if kept separate for events from `InstrumentationRuntime`):
*   `correlate_event(bridge_ref :: pid() | atom(), event :: map()) :: :ok` (Called by `InstrumentationRuntime.ASTReporting.maybe_forward_to_temporal_bridge`)

## 5. Core Data Structures

Defined in `ElixirScope.TemporalDebug.Types`:

*   **`ast_enhanced_state()`**:
    ```elixir
    %{
      original_state: map(), # State from basic event replay
      ast_context: ElixirScope.Correlator.Types.ast_context() | nil, # CPG context at this timestamp
      structural_info: map(), # e.g., current CFG block, call depth in CPG
      active_cpg_path: list(String.t()), # Sequence of CPG nodes leading to this state
      variable_flow_at_timestamp: map(), # {var_name -> {value, defining_cpg_node_id}}
      timestamp: non_neg_integer()
    }
    ```
*   **`ast_execution_trace()`**:
    ```elixir
    %{
      trace_id: String.t(),
      events_with_cpg_context: list(ElixirScope.Correlator.Types.enhanced_event()),
      cpg_node_sequence: list(%{cpg_node_id: String.t(), cpg_node_label: String.t(), timestamp: integer()}),
      state_snapshots_with_cpg_context: list(ast_enhanced_state()), # Snapshots at key CPG transitions
      metadata: map() %{session_id: String.t(), start_time: integer(), end_time: integer()}
    }
    ```
*   Consumes: `ElixirScope.Events.t()` (from `elixir_scope_storage`/`TemporalStorage`).
*   Consumes: `ElixirScope.Correlator.Types.ast_context()` (from `elixir_scope_correlator`).
*   Consumes: CPG details (indirectly via `elixir_scope_correlator` from `elixir_scope_ast_repo`).

## 6. Dependencies

This library will depend on the following ElixirScope libraries:

*   `elixir_scope_utils` (for utilities).
*   `elixir_scope_config` (for its operational parameters, cache settings).
*   `elixir_scope_events` (to understand event structures).
*   `elixir_scope_storage` (or its own `TemporalStorage` needs to interface with the main event stream for raw event data).
*   `elixir_scope_correlator` (CRUCIAL for getting CPG context for events and states).
*   `elixir_scope_ast_repo` (indirectly via `elixir_scope_correlator` for CPG details).
*   `elixir_scope_ast_structures` (for CPG related type definitions).

It will depend on Elixir core libraries (`GenServer`, `:ets`).

## 7. Role in TidewaveScope & Interactions

Within the `TidewaveScope` ecosystem, the `elixir_scope_temporal_debug` library will:

*   Be a key service providing the backend for "Execution Cinema" features.
*   Its API will be exposed via `TidewaveScope` MCP tools, allowing an AI assistant to:
    *   "Reconstruct the state of process P at time T, and show me which CPG node was active."
    *   "Show the CPG execution trace for the web request with correlation ID C."
    *   "Find all times module M, function F (CPG node N) was executed and show the CPG context."
*   The "Cinema Debugger UI" (a separate project) would heavily consume the APIs of this library to visualize time-travel and CPG-aware execution flows.

## 8. Future Considerations & CPG Enhancements

*   **Differential State Reconstruction:** Optimize state reconstruction by only replaying deltas from the nearest stored snapshot, enhanced with CPG context changes.
*   **CPG-Path Based Navigation:** Allow "stepping" through execution history by CPG edges (control flow, data flow, call graph) rather than just chronological events.
*   **Querying Past States with CPG Predicates:** "Find all past states where variable `X` (at CPG def-site `D`) had value `V` AND execution was within a CPG node matching pattern `P`."
*   **Visual Trace-to-CPG Mapping:** The data structures from this library are designed to make it easier for a UI to visually map an execution trace directly onto a CPG visualization.
*   **Performance Optimization for Large Traces:** Strategies for handling and querying very long execution histories, potentially involving summarization or more advanced indexing linked to CPG regions.

## 9. Testing Strategy

*   **`ElixirScope.TemporalDebug.TemporalStorage` Unit Tests:**
    *   Test storing and retrieving events by time range, `ast_node_id`, `correlation_id`.
    *   Test temporal ordering.
    *   Test pruning and stats.
*   **`ElixirScope.TemporalDebug.TemporalBridgeEnhancement` (and its submodules) Unit Tests:**
    *   **`StateManager`**:
        *   Mock `TemporalStorage` to provide event sequences.
        *   Mock `elixir_scope_correlator` to provide `ASTContext` for events/timestamps.
        *   Verify `reconstruct_state_with_ast` produces the correct `ast_enhanced_state` (correct original state combined with correct CPG context).
    *   **`TraceBuilder`**:
        *   Provide mock sequences of `EnhancedEvent`s (from a mock `elixir_scope_correlator`).
        *   Verify `get_ast_execution_trace` produces traces with correct CPG node sequences.
    *   **`EventProcessor`**: Test event filtering and querying logic against a mock `TemporalStorage`.
    *   **`ASTContextBuilder`**: Test its interaction with a mock `elixir_scope_correlator`.
    *   **`CacheManager`**: Test caching of states and traces.
*   **Integration Tests:**
    *   Test the full flow: events written to `TemporalStorage` -> `TemporalBridgeEnhancement` reconstructs state -> verify CPG context is correctly fetched from a mock `elixir_scope_correlator` / `elixir_scope_ast_repo`.
    *   Test time-travel queries returning expected data enriched with CPG context.
*   **Performance Tests:**
    *   Benchmark state reconstruction time with varying numbers of events and CPG complexity.
    *   Benchmark trace generation time.
