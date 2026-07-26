// INPUT:
//   concession_reports — Structured concession data per archival project (concession_required,
//                         provision_of_DCPR, approval_req_from, justification_by_LS,
//                         comments_by_AE, comments_by_EE, combined into a single text block),
//                         produced by parsing raw PDF concession reports
//   query_bank          — 41 supplementary-information queries, each linked to one of the
//                         26 retained supplementary-information concepts (see
//                         concept-extraction.md) and grouped under a category (undertakings,
//                         NOCs, promises, registered personnel, subjective judgements,
//                         design parameters)
//
// OUTPUT: supplementary_info_store — a table of {1 (True), 0 (False), None} answers to all
//         41 queries, one row per archival project. At the application stage, predicted
//         values for a new project are obtained by voting across the top 3 retrieved
//         projects' rows of this table (see voting.md).

FUNCTION Extract_Supplementary_Information(concession_reports, query_bank):

    // =========================================================
    // 1. QUERY PROMPT ASSEMBLY
    // =========================================================
    // Format all 41 queries into a single numbered block for a one-shot extraction
    // prompt (one LLM call answers all 41 queries per project)
    query_prompt_block = ""
    FOR EACH query IN query_bank (IN QUERY ID ORDER):
        query_prompt_block += f"{query.id}: {query.text}\n"

    // =========================================================
    // 2. PER-PROJECT EXTRACTION (LLM)
    // =========================================================
    supplementary_info_store = INITIALIZE_EMPTY_TABLE()

    FOR EACH report IN concession_reports:
        prompt = BUILD_EXTRACTION_PROMPT(context = report.structured_text,
                                          questions = query_prompt_block)
        // system instruction: answer strictly in JSON as {query_id: boolean | null};
        // return null when the query cannot be answered from the given context
        response = LLM_EXTRACT_ANSWERS(prompt, response_schema = QUERY_ANSWER_SCHEMA,
                                        model = "GPT-4o")

        answer_row = INITIALIZE_EMPTY_ROW()
        FOR EACH (query_id, answer) IN response.all_answers:
            answer_row[query_id] = answer   // answer IN {1 (True), 0 (False), None}

        supplementary_info_store[report.id] = answer_row

    // =========================================================
    // 3. SAVE SUPPLEMENTARY INFORMATION STORE
    // =========================================================
    SAVE_AS_EXCEL(supplementary_info_store, "combined_query_answers.xlsx")

    RETURN supplementary_info_store
