// INPUT:
//   new_project_scrutiny_report — HTML scrutiny report for the incoming project requiring
//                                  a concession justification
//   vector_database              — Feature-vector store built during the development stage
//                                  from the archival (database) projects by
//                                  Process_Scrutiny_Reports (see feature-engineering.md).
//                                  Includes, per archival project, its processed feature
//                                  vector and project ID, plus the fitted schema needed to
//                                  encode a new query consistently: feature_names,
//                                  categorical_columns, trained one-hot encoder, trained
//                                  KNN imputer, and scaling parameters.
//   concession_metadata_store    — Structured concession information per archival project
//                                  (concession_required, provision_of_DCPR, approval_req_from,
//                                  justification_by_LS, comments_by_AE, comments_by_EE)
//   dcpr_bounds                  — as in feature-engineering.md
//
// OUTPUT: top_projects — the top 3 archival projects, rank-ordered by cosine similarity,
//         each with its similarity score and structured concession metadata; passed
//         downstream to supplementary-information prediction (voting.md) and to prompt
//         assembly for generation (generation.md)

FUNCTION Retrieve_Similar_Projects(new_project_scrutiny_report, vector_database,
                                    concession_metadata_store, dcpr_bounds):

    // =========================================================
    // 1. QUERY VECTOR CONSTRUCTION
    // =========================================================
    // Derive the new project's structured feature representation using the same
    // concept-guided feature engineering applied to the archival database, so the
    // query vector is directly comparable (see feature-engineering.md).

    query_features = Process_Scrutiny_Reports([new_project_scrutiny_report], dcpr_bounds)
                      // single-row feature vector, before schema alignment below

    // --- Schema alignment ---
    // Encode categorical columns using the encoder fitted on the archival database
    // (transform only, not refit), and align to the database's saved column schema
    query_features = APPLY_ONE_HOT_ENCODING(query_features, vector_database.categorical_columns,
                                             encoder = vector_database.ohe)
    query_features = ALIGN_COLUMNS(query_features, vector_database.feature_names, fill_value = 0)

    // --- Missing-value handling ---
    // Consistent with how the archival database itself was built: continuous columns
    // are imputed with the KNN imputer fitted on the archival database; remaining
    // missing values default to 0
    query_features = APPLY_TRAINED_KNN_IMPUTER(query_features, vector_database.continuous_imputer)
    query_features = FILL_MISSING(query_features, value = 0)

    // --- Feature scaling ---
    // Apply the min-max scaling parameters fitted during development (see
    // feature-engineering.md), so the query vector is on the same scale as
    // vector_database
    query_features = APPLY_MIN_MAX_SCALING(query_features, vector_database.scaling_parameters)

    // =========================================================
    // 2. SIMILARITY SEARCH
    // =========================================================
    similarity_scores = INITIALIZE_EMPTY_LIST()

    FOR EACH archival_project IN vector_database.projects:
        score = COSINE_SIMILARITY(query_features, archival_project.feature_vector)
        similarity_scores.APPEND( (archival_project.id, score) )

    // =========================================================
    // 3. TOP-K SELECTION
    // =========================================================
    // Rank archival projects by similarity score, descending, and retain the
    // top three unique projects.

    ranked_projects = SORT_DESCENDING(similarity_scores, key = score)
    top_projects = ranked_projects[0:3]
    // top_projects: [(project_id, similarity_score), ...] in similarity-rank order

    // =========================================================
    // 4. METADATA RETRIEVAL
    // =========================================================
    // For each retrieved project, attach its structured concession information,
    // preserving similarity-rank order.

    retrieved_projects = INITIALIZE_EMPTY_LIST()

    FOR EACH rank, (project_id, score) IN ENUMERATE(top_projects):
        concession_data = concession_metadata_store.LOOKUP(project_id)
                            // concession_required, provision_of_DCPR, approval_req_from,
                            // justification_by_LS, comments_by_AE, comments_by_EE

        retrieved_projects.APPEND({
            'project_id':       project_id,
            'similarity_score': score,
            'rank':              rank,
            'concession_data':   concession_data
        })

    // =========================================================
    // 5. RETURN RETRIEVED PROJECTS
    // =========================================================
    RETURN retrieved_projects
