// INPUT:
//   archival_concession_reports — Raw PDF concession reports (unstructured), one per
//                                  archival project (300 development-set projects; 33
//                                  held-out test-set projects evaluated separately)
//   archival_scrutiny_reports   — Corresponding HTML scrutiny reports, one per archival
//                                  project
//   dcpr_bounds                  — Lookup table of DCPR-prescribed regulatory limits
//                                  (see feature-engineering.md)
//   query_bank                   — 41 supplementary-information queries (see
//                                  supplementary-information-prediction.md)
//   optional_query_bank          — Free-text clarifying questions for the selected
//                                  concession type (see hitl-verification.md)
//   new_project_scrutiny_report — HTML scrutiny report for the new project requiring a
//                                  concession justification (application stage)
//   llm_model                    — GPT-4o
//
// OUTPUT: generated_concession_justification for the new project (application stage).
//         The development stage additionally persists vector_database,
//         concession_metadata_store, and supplementary_info_store for reuse across all
//         future application-stage runs, so it is normally executed once, not per project.
//
// This function sequences the pseudo-coded stages of the CG-RAG with HITL framework
// into the two-stage structure shown in the paper's Fig. 3: a development flow that
// builds structured representations of archival projects, and an application flow
// that processes a new project through retrieval, prediction, verification, and
// generation.

FUNCTION CG_RAG_With_HITL(archival_concession_reports, archival_scrutiny_reports, dcpr_bounds,
                          query_bank, optional_query_bank, new_project_scrutiny_report, llm_model):

    // =========================================================================
    // DEVELOPMENT STAGE — build structured representations of archival projects
    // (run once; re-run only when new archival projects are added to the database)
    // =========================================================================

    // --- 0. Structured concession parsing (LLM) ---
    // Normalize each archival concession report (PDF -> plain text, with OCR-preserved
    // table structure) and extract structured fields via LLM: concession_required,
    // provision_of_DCPR, approval_req_from, justification_by_LS, comments_by_AE/EE.
    concession_metadata_store = Parse_Concession_Reports(archival_concession_reports)

    // --- 1. Concept identification and classification ---
    // Identify recurring regulatory/reasoning concepts across archival concession
    // reports, disambiguate, track saturation, and classify into feature-engineering
    // concepts (24) vs. supplementary-information concepts (26); derived data and
    // case-specific data are pruned out.
    feature_concepts, supplementary_concepts =
        Extract_And_Classify_Concepts(concession_metadata_store, archival_scrutiny_reports,
                                       dcpr_bounds, min_occurrence_count)
        // see concept-extraction.md

    // --- 2. Feature construction (24 concepts -> 49 features) ---
    // Build the structured, scaled feature vector for every archival project from
    // its scrutiny report. The resulting matrix is the vector_database used for
    // similarity-based retrieval.
    vector_database = Process_Scrutiny_Reports(archival_scrutiny_reports, dcpr_bounds)
        // see feature-engineering.md

    // --- 3. Supplementary-information extraction (41 queries, LLM) ---
    // For each archival project, answer all 41 supplementary-information queries
    // from its structured concession text.
    supplementary_info_store =
        Extract_Supplementary_Information(concession_metadata_store, query_bank)
        // see supplementary-information-prediction.md

    SAVE(vector_database, concession_metadata_store, supplementary_info_store)

    // =========================================================================
    // APPLICATION STAGE — process one new project (run per incoming project)
    // =========================================================================

    // --- 4. Knowledge retrieval ---
    // Build the new project's feature vector (schema-aligned to vector_database)
    // and retrieve the top 3 most similar archival projects, along with each
    // one's structured concession metadata.
    top_projects = Retrieve_Similar_Projects(new_project_scrutiny_report, vector_database,
                                              concession_metadata_store, dcpr_bounds)
        // see retrieval.md

    // --- 5. Supplementary-information prediction (voting) ---
    // Predict the new project's 41 supplementary-information values by voting,
    // per query, across the top 3 retrieved projects' stored labels.
    predicted_supplementary_info =
        Aggregate_Supplementary_Information(top_projects, supplementary_info_store)
        // see voting.md

    // --- 6. Human-in-the-loop verification ---
    // Present the predicted values, the retrieved projects, and the optional
    // free-text questions to the user; capture review/overrides and any additional
    // project-specific input before generation.
    verified_context = HITL_Verify_Context(predicted_supplementary_info, query_bank,
                                            top_projects, optional_query_bank)
        // see hitl-verification.md

    // --- 7. Generation ---
    // Assemble the verified context, the current project's feature summary, and
    // the retrieved precedents' full concession text into a prompt, and generate
    // the concession justification.
    curated_project_summary = Curate_Project_Summary(new_project_scrutiny_report)
        // natural-language serialization of the 24 concept-guided feature values
        // for the current project (see feature-engineering.md); constructed
        // alongside the query vector in retrieval.md Step 1

    generated_concession_justification =
        Generate_Concession_Justification(verified_context, curated_project_summary, llm_model)
        // see generation.md

    RETURN generated_concession_justification
