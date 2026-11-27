# Part 3: Build and run the legal research agent

In this exercise, you will build the agent one tool at a time, test each addition, and compare its answers as it gains capabilities. You will then add memory, use the chat interface, and test failover.

## Before you begin

1. Ensure you have completed **Part 2**, including pipeline verification, citation-graph loading, and both search indexes.
1. Open **2-app-development.ipynb** in VS Code and select the lab **Python 3.11.9** kernel.
1. Keep this guide beside the notebook. Each code cell has a visible label such as **Task 1 Cell 1**. Cell numbering starts at 1 within each task.

    > [!IMPORTANT]
    > Run the named cells in task order, checking each result before continuing. Do not use **Run All**: the memory exercise requires several turns, and the chat interface must be running before the failover test.

## Task 1: Check dependencies and load the agent configuration

You will confirm that the selected interpreter can load the required libraries, then configure the application-side chat client and database connection. This avoids discovering a package or credential problem during an agent run.

1. Run **Task 1 Cell 1**, beginning with *from pathlib import Path*, before the third-party imports.
1. Confirm *Dependency preflight passed.* If it fails, run the printed repair command in the VS Code terminal, **restart the notebook kernel**, and rerun Task 1 Cell 1.
1. Run **Task 1 Cell 2**, beginning *import os*, to load the application imports.
1. Run **Task 1 Cell 3**, beginning *env_path = requirements_path.with_name(".env")*.
1. Confirm the chat deployment matches your lab and that the semantic sink is *public.case_opinion_chunks*. The managed aliases must be *lab-embedding* and *lab-chat*.

    > [!NOTE]
    > Microsoft Agent Framework runs the chat-and-tool loop in Python. Database-side embeddings and fact extraction use the registered HorizonDB model aliases; application chat settings are loaded separately from **src/.env**.

## Task 2: Test keyword search and build your first agent

You will define keyword retrieval, call it directly, and give it to a one-tool agent. The goal is to see the difference between raw search results and a model-generated answer based on those results.

1. Run **Task 2 Cell 1**, containing *def keyword_case_search(`*. Read its *@tool* description, which tells the agent to search using doctrine names or distinctive terms.
1. Run **Task 2 Cell 2**, the direct keyword-search test. Inspect the case IDs, opinion snippets, and *score=* values in the returned rows.
1. Run **Task 2 Cell 3** to build and invoke the one-tool agent with the supplied question.
1. Confirm *keyword_case_search was called* appears in the trace. Read the agent's answer and compare its named cases with the raw results from Task 2 Cell 2.

    > [!NOTE]
    > BM25 ranks exact-term matches in full opinions; lower scores mean stronger matches in these results. Calling the tool directly checks retrieval, while the agent run shows how the model selects arguments and turns retrieved evidence into an answer.

## Task 3: Add semantic search and compare hybrid results

You will add semantic retrieval to find cases with similar facts, even when their wording differs. You will compare the two result sets before letting a two-tool agent combine them.

1. Run **Task 3 Cell 1**, beginning with *VALID_COURT_LEVELS =*, to define *semantic_case_search*.
1. Note its required *court_level* and *min_year* arguments. *Both* and *1900* provide broad retrieval while exercising the filters.
1. Run **Task 3 Cell 2** to call keyword and semantic search directly and combine their case IDs.
1. Compare the keyword and semantic ID lists. Identify one overlapping case and, if present, one case found by only one method.
1. Inspect *distance=* values in semantic results and confirm *advanced_filtering=ON* appears in the trace.
1. Run **Task 3 Cell 3** to assemble and invoke the two-tool agent.
1. Confirm both retrieval tools were called. Compare the answer with the one-tool answer from Task 2: what additional evidence did semantic search contribute?

    > [!NOTE]
    > DiskANN searches chunk embeddings and retains each case's best matching chunk. Keyword and semantic results complement each other; their combined case IDs become the input to graph expansion in the next task.

## Task 4: Expand the candidates through the citation graph

You will add graph retrieval to find precedents connected to the candidate cases. The goal is to move beyond similar text and include cases linked through citations.

1. Run **Task 4 Cell 1**, containing *def precedent_graph_search(`*.
1. Read its input description: the tool expects the combined case IDs from keyword and semantic retrieval.
1. Run **Task 4 Cell 2** to test the three retrieval tools together.
1. Compare *Input ids* with *RELATED_CASE_IDS*. The graph results should expand the inputs rather than return the same cases.
1. Inspect the path counts and ranking. *Washington Supreme Court* cases receive a score boost of 2.
1. Run **Task 4 Cell 3** to assemble and invoke the three-tool agent.
1. Confirm all three tools appear in the trace. Compare its cited precedents with the raw graph results and the two-tool answer from Task 3.

    > [!NOTE]
    > Apache AGE follows **REF** citations up to two hops in both directions. Path counts and the court-level boost help rank related authority; the graph's case IDs preserve the connection to authoritative opinions in **public.cases**.

## Task 5: Extract legal facts before composing the answer

You will add a tool that reads full opinions and extracts holdings, issues, statutes, and dispositions. The goal is a brief grounded in structured legal facts rather than short search snippets alone.

1. Run **Task 5 Cell 1**, containing *def case_analyst_extract(`*.
1. Locate its extraction fields: *holding*, *issues*, *statutes_cited*, and *disposition*. Note that its case IDs should come from the graph tool's *RELATED_CASE_IDS* output.
1. Run **Task 5 Cell 2** to assemble and invoke the four-tool agent.
1. Confirm the trace includes the three retrieval tools and *case_analyst_extract was called*, followed by *extracted_case=* lines.
1. Inspect any extraction errors before accepting the response. Missing or failed extraction is not evidence that a case has no relevant holding.
1. Compare the answer with Task 4. Identify a holding, issue, or statute now discussed in more detail.

    > [!NOTE]
    > **azure_ai.extract** reads full opinions inside HorizonDB using **lab-chat**. Extraction enriches the graph-selected cases, but its model-generated fields still need checking against the source opinions.

## Task 6: Retrieve and inspect weather evidence

You will define an external evidence tool and test it directly before giving it to the agent. This separates API and parsing problems from the model's decision about when to call the tool.

1. Run **Task 6 Cell 1**, containing *def get_weather_evidence(`*.
1. Read its required arguments: latitude, longitude, and start date. The end date is optional.
1. Run **Task 6 Cell 2**, the direct weather test using Seattle coordinates and the supplied January 2023 date range.
1. Inspect **WEATHER_EVIDENCE_JSON**, its period, daily records, total precipitation, and days with rain. Check that the totals agree with the daily values.
1. If the tool returns an error instead of JSON, check the message and outbound access to **archive-api.open-meteo.com** before continuing.

    > [!NOTE]
    > Unlike the database tools, this tool calls Open-Meteo's public archive API. Its description tells the agent to use it for weather-related questions, not every legal question.

## Task 7: Run the complete five-tool agent

You will submit a Seattle flooding scenario to an agent with all five tools. The goal is to combine case law, structured legal facts, and rainfall evidence into one draft brief.

1. Locate **Task 7 Cell 1**, beginning with *client = OpenAIChatClient(*.
1. Check that *tools=[...]* lists all five functions defined in Tasks 2 through 6.
1. Read the supplied *user_query*: a client's Seattle property floods after the city changed road drainage in **February 2024**. Leave it unchanged for this run.
1. Run **Task 7 Cell 1** once. Wait for the tool trace and the final *// Agent Response: //*.
1. Check that all five tools were called, semantic retrieval reports *advanced_filtering=ON*, and extraction reports *extracted_case=* lines.
1. Check that weather retrieval uses Seattle coordinates, approximately *47.6062, -122.3321*, and February 2024 dates. These differ from the direct test in Task 6.
1. If a tool fails or is not called, inspect the error, tool description, and agent instructions before treating the response as a completed five-tool result.

    > [!NOTE]
    > Agent Framework executes the tool calls the model requests. The expected flow is keyword and semantic retrieval, combined case IDs, graph expansion, and legal extraction, with weather evidence supporting the factual analysis.

## Task 8: Check the brief against the evidence

You will review the response rather than judging success by fluent wording alone. The goal is to verify that adding the fifth tool improved the brief with actual weather evidence.

1. Confirm the trace includes all five tools and no unresolved extraction or weather errors.
1. Check that the response names cases and discusses holdings, issues, and dispositions. Statutes should appear when the extracted evidence provides them.
1. Compare the brief with the four-tool answer from Task 5. Check that rainfall totals and dates come from the weather output, rather than being invented.
1. Identify one claim that still needs checking against a full opinion or external evidence. Share it with the facilitator or record it for follow-up.

    > [!IMPORTANT]
    > This is a research proof of concept, not legal advice. A tool-call trace demonstrates execution, not factual correctness; verify cited sources and model-generated claims before relying on them.

## Task 9: Add memory and test recall across turns

You will store client facts in HorizonDB with Mem0 and retrieve them in later turns. The goal is to see how persistent memory changes a previously stateless agent.

1. Locate **Task 9 Cell 1**. Keep exactly one supplied *user_query* block uncommented at a time.
1. Run **Task 9 Cell 1** with the first block, the intake turn. Inspect the retrieved and newly extracted memories.
1. Comment out the first block, uncomment the second, and rerun Task 9 Cell 1. Confirm the intake facts appear in retrieved memory.
1. Repeat with the third block, the follow-up. Check that the response uses facts or legal context from the previous turn.
1. Repeat with the fourth block, the recall test. Confirm the agent recalls the neighborhood and storm month without you repeating them.

    > [!NOTE]
    > Mem0 uses pgvector in HorizonDB to store and retrieve facts scoped by user ID. Previously stored facts may already appear on the first turn if that lab user has run the exercise before.

## Task 10: Use the chat interface with memory

You will launch the Gradio interface and repeat a short conversation through the browser. The goal is to observe tool calls and long-term memory alongside the agent's response.

1. After Task 9 Cell 1 succeeds, run **Task 10 Cell 1** to launch Gradio. Wait for **Running on local URL: http://127.0.0.1:7860**.
1. Open `http://localhost:7860` in a new tab in the VM browser, send the flooding scenario, and inspect the **Tool Trace** and **Long-term Memory** panels.
1. Send a follow-up without repeating the location or storm date. Confirm the agent recalls them.
1. Leave the app running for Task 11.

    > [!NOTE]
    > The interface uses the memory client initialized in Task 9 Cell 1. It has its own user ID, separate from the notebook conversation, and clears that UI user's memories each time Task 10 Cell 1 is rerun.

## Task 11: Observe the app during a lab database failover

You will trigger failover on your assigned lab instance and check how the running app recovers. The goal is to distinguish a brief connection interruption from loss of stored client memory.

1. Keep Gradio running and send a baseline chat turn. Perform this test only on your assigned lab HorizonDB instance, not a shared or production database.
1. Open a new tab and navigate to `https://portal.azure.com`, open the lab HorizonDB resource and select **Settings** > **High Availability**.
1. Select **Failover** and confirm **Fail over**.
1. Send another chat turn and inspect notebook output for Mem0 retry messages. A turn during the interruption can fail; investigate any tool or connection error.
1. After failover completes, send a follow-up and confirm that memories are retrieved again.
1. Shut down the notebook kernel when finished with the app.

    > [!NOTE]
    > Failover interrupts existing connections. The lab tools open a new connection per call, while Mem0 uses a short retry helper. This demonstrates recovery behavior, not a complete production retry policy.

## Next Steps

> Congratulations! You have completed all the lab exercises and reached the end of the lab.
