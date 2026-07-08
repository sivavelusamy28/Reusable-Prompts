Analyze the currently open Java file in depth. Treat this file as the primary 
unit of analysis — only look outside it (other classes/methods in the workspace) 
when this file calls, extends, implements, or depends on something external and 
understanding that dependency is necessary to judge correctness or risk. Do not 
review or critique the external code itself — just note the interaction.

Cover these areas:

1. RESILIENCY
   - Exception handling: are exceptions caught at the right granularity, or is 
     there overly broad catch(Exception)/catch(Throwable)?
   - Are checked exceptions swallowed, logged-and-ignored, or rethrown without context?
   - Retry logic: is it present where transient failures are likely (DB calls, 
     Kafka, HTTP, I/O)? Is backoff/jitter used or is it a tight loop?
   - Timeouts: are they configured for external calls (DB connections, HTTP clients, 
     Kafka consumers/producers)? Any hardcoded/missing timeouts?
   - Resource management: try-with-resources used correctly for Closeable/AutoCloseable? 
     Any connection/stream leaks on exception paths?
   - Null safety: Optional usage vs raw null checks; any NPE risk on external inputs?
   - Thread safety: shared mutable state, missing synchronization, unsafe use of 
     CompletableFuture/ExecutorService (e.g., unbounded thread pools, no exception 
     handling in async chains)?
   - Idempotency: for methods that mutate state (DB writes, Kafka produces), is 
     duplicate-processing handled?
   - Circuit breaker / bulkhead patterns: present where warranted, or should be 
     considered?
   - Fail-fast vs fail-safe: does the code degrade gracefully or cascade failures?

2. CODE STRUCTURE
   - Indentation and formatting consistency (spaces vs tabs, brace placement, 
     line length)
   - Method length and single-responsibility violations
   - Naming conventions (Java standard: camelCase, meaningful names, no abbreviations 
     that hurt readability)

3. COMMENTS & DOCUMENTATION
   - Are Javadoc comments present on public methods/classes? Are they accurate 
     and not stale relative to the code?
   - Are inline comments explaining "why" (useful) vs "what" (redundant/noise)?
   - Any missing comments where business logic or a non-obvious workaround needs 
     explanation (e.g., DB2 -803 handling, Kafka offset logic, custom retry logic)?
   - Any commented-out dead code that should be removed?

4. OUTPUT FORMAT
   For each finding, provide:
   - Location (method name + approximate line number)
   - Issue category (Resiliency / Formatting / Comments)
   - Severity (High / Medium / Low)
   - Specific suggested fix (code snippet where useful, not just a description)

   End with a short summary table: total issues found, grouped by category and severity.

Do not rewrite the entire file. Only show diffs/snippets for the specific lines 
that need to change.
