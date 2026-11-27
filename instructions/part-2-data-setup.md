# Part 2: Prepare the legal research data

In this exercise, you will load court cases into HorizonDB and prepare keyword, vector, and citation-graph retrieval for your agent. You will test each feature before building on it.

## Before you begin

1. Complete [Part 0](0-sign-in.md) and [Part 1](1-connect.md). Use the provisioned lab VM, HorizonDB instance, and Microsoft Foundry deployments.
1. In VS Code, open the lab repository. On the lab VM, the folder is `C:\Users\LabUsers\source\repos\frontiertour-ILL392`.
1. Keep this guide open beside the notebook. Each code cell has a visible label such as **Task 1 Cell 1**. Cell numbering starts at 1 within each task.

> [!IMPORTANT]
> Run the named cells in task order, waiting for each to finish and checking its output.

## Task 1: Open the notebook and connect to your environment

You will select the lab Python kernel and load the database and model settings. The goal is to confirm that the notebook is using your provisioned environment before creating any data.

1. In the VS Code **Explorer**, expand **code** and open [1-data-setup.ipynb](../code/1-data-setup.ipynb).
1. Select **Select Kernel** and choose the lab **Python 3.11.9** interpreter.

    ![Select the Python kernel](../img/1.9-python-runtime.png)

1. Run **Task 1 Cell 1**, beginning `%pip install -r ../src/requirements.txt`, using **Run Cell**. Wait for it to finish.
1. Run **Task 1 Cell 2**, beginning `import os`, to load the imports.
1. Run **Task 1 Cell 3**, beginning `env_path = next(`, to load the environment.
1. Check that the printed database, Azure OpenAI endpoint, and deployment names match your lab resources. The database model API version should print as `None`.

> [!NOTE]
> Connection settings are loaded from `src/.env`; passwords and API keys should never be pasted into notebook cells. If a setting is missing, compare the file with `src/.env.sample` and use the lab **Resources** tab to find your assigned credentials.

## Task 2: Enable database AI features and register the models

You will verify the required PostgreSQL extensions, register the chat and embedding deployments, and test both models. Later database queries will use the short aliases `lab-chat` and `lab-embedding` instead of carrying model credentials.

1. Run **Task 2 Cell 1**, beginning `REQUIRED_EXTENSIONS = [`.
1. Confirm success for all five required extensions: `azure_ai`, `vector`, `age`, `pg_diskann`, and `pg_textsearch`. `pg_durable` is optional.
1. If an extension check fails, follow the printed parameter-group correction and wait for any restart before continuing.
1. Run **Task 2 Cell 2**, beginning `CHAT_MODEL_ALIAS = "lab-chat"`.
1. Confirm both model aliases are registered and their verification rows show the assigned endpoint and deployments.
1. Run **Task 2 Cell 3**, the text-generation test using `lab-chat`. Confirm it prints generated text without an error.
1. Run **Task 2 Cell 4**, the single-row embedding test using `lab-embedding`. Inspect the returned vector and confirm the call succeeds.
1. Compare the two outputs: the chat model returns readable text, while the embedding model returns numeric values used for similarity search.

> [!NOTE]
> `azure_ai` makes registered AI models available inside SQL. The other extensions add vector storage, DiskANN similarity search, BM25 keyword search, and Apache AGE graph traversal to the same database.

## Task 3: Load the case dataset

You will load the supplied court cases into typed PostgreSQL columns. This creates the authoritative case records used by keyword search, graph retrieval, and legal fact extraction.

1. Run **Task 3 Cell 1**, beginning `csv_path = env_path.parent.parent / "data" / "cases.csv"`.
1. Confirm that the printed CSV path points to this repository's `data\cases.csv`.
1. Wait for `Done. <N> cases loaded into the cases table.` Confirm that the count is greater than zero.
1. Run **Task 3 Cell 2**, beginning `conn = psycopg.connect(**DB_CONFIG)`, to preview the loaded cases.
1. Inspect the five returned cases. Locate their case IDs, dates, court names, and opinion text before using these fields in the pipeline.

> [!NOTE]
> The load uses PostgreSQL `COPY` to stream the CSV, then SQL to extract case fields from JSON. `public.cases` holds the full opinions; `public.temp_cases` retains the raw citation data needed for the graph.

## Task 4: Create, monitor, and verify the embedding pipeline

You will split opinions into chunks, generate their embeddings inside HorizonDB, and link the results back to the correct cases. The goal is a verified vector-search dataset, not just a successful pipeline status.

1. Run **Task 4 Cell 1**, beginning `conn = psycopg.connect(**DB_CONFIG)`, to create the pipeline sink.
1. Confirm that all seven sink columns are printed: `doc_id`, `chunk_index`, `chunk_text`, `embedding`, `metadata`, `court_level`, and `decision_date`.
1. Run **Task 4 Cell 2**, beginning `POLL_SECONDS = 10`. Watch the chunk and embedding counts grow, and wait for `last_run_status` to become `completed` or `succeeded`.
1. While the pipeline runs, open the **PostgreSQL** extension. Right-click the lab database, select **Pipelines & Workflows**, and open **AI Pipelines**.
1. Select `case_opinion_embedding_pipeline`. Confirm the graph shows the case source, chunk step, embedding step, and chunk-table sink. Inspect any failed node before continuing.
1. After Task 4 Cell 2 completes, run **Task 4 Cell 3**, beginning `def enrich_pipeline_output(cur):`. Wait for `Done. Pipeline completed and all chunks enriched.`
1. Run **Task 4 Cell 4**, beginning `checks = []`. Confirm **every invariant passes**, including source coverage, nonempty text, 1536-dimension embeddings, valid case IDs, and matching court/date metadata.
1. Inspect the returned semantic matches. **Do not continue to indexing or Part 3 if any check fails.**

> [!NOTE]
> The preview chunk step emits internal document IDs rather than case IDs. Enrichment checks each mapping and copies court/date fields beside the vectors so DiskANN can filter during retrieval. `public.case_opinion_chunks` owns chunk text and embeddings; `public.cases` remains authoritative for full opinions.

> [!IMPORTANT]
> Task 4 Cell 1 resets the sink. If the pipeline times out, inspect its status and wait; do not reset an active run. If only enrichment fails after completion, retry Task 4 Cell 3 against the existing chunks instead of regenerating embeddings. Use [diagnostics](4-diagnostics.md) to investigate failed checks.

## Task 5: Load the citation graph

You will turn the dataset's citation relationships into an Apache AGE graph. Your agent will use this graph to expand retrieved cases into related precedents.

1. Run **Task 5 Cell 1**, beginning `import time`, to load the citation graph.
1. Watch the `case_graph_load` run status and processed-row output. Wait for the run to complete.
1. Confirm the final message reports `case_graph is ready` with case-node and unique `REF`-edge counts. The cell checks these counts against the graph source.
1. If the run fails, times out, or reports changed source facts, stop and inspect its error before rerunning; do not reset the graph to bypass the check.

> [!NOTE]
> A `case` node represents a court case, and an outgoing `REF` edge means that case cites another. This manual, append-only graph pipeline uses the supplied citation data rather than asking a model to infer relationships.

## Task 6: Build the keyword and vector indexes

You will index full opinions for exact-term retrieval and opinion chunks for semantic retrieval. Both indexes must be ready before the agent uses them.

1. Run **Task 6 Cell 1**, beginning `conn = psycopg.connect(**DB_CONFIG)`, to build the keyword index.
1. Confirm `BM25 index created on cases.opinion` and inspect the ranked results for `water leaking`.
1. Run **Task 6 Cell 2**, beginning `DISKANN_INDEX_NAME = "idx_case_opinion_chunks_diskann"`.
1. Confirm `idx_case_opinion_chunks_diskann` is valid and ready. Either the supported quantized build or the full-precision build is acceptable.
1. Confirm Tasks 4 and 5 completed without failed checks before exploring the database in Task 7.

> [!NOTE]
> BM25 finds distinctive words in full opinions; DiskANN finds semantically similar chunks. Court and date filters live on the chunk table so vector retrieval can apply them during the index scan.

## Task 7: Explore the schema and citation graph

You will inspect the objects you just created in the PostgreSQL extension. The goal is to connect the code-cell outputs to the actual tables, vectors, and citation relationships your agent will query.

1. In the PostgreSQL extension, right-click the database and select **Visualize Schema**. Compare `public.cases` with `public.case_opinion_chunks`.
1. Find the case ID in the source table and `doc_id` in the chunk table. Notice that one case can have several chunks.
1. Open [show_graph.sql](show_graph.sql) in a PostgreSQL query window, run it, and select the graph visualization to explore citations.
1. Select a case node and follow a `REF` edge to a cited case. Then open [Part 3](3-app-development.md).

> [!NOTE]
> Tables, vector indexes, and the AGE graph share one HorizonDB database. Their case IDs let the application combine evidence across retrieval methods without a separate vector or graph store.

## Next Steps

> Select **Next >** to proceed to the next part of the lab.
