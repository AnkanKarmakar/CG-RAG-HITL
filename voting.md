// INPUT:
//   retrieved_projects          — Top-k retrieved archival projects for a target project,
//                                 ordered by descending similarity score
//   supplementary_label_matrix  — Matrix containing extracted supplementary-information
//                                 labels for archival projects
//                                 rows    = archival projects
//                                 columns = 41 supplementary-information queries
//                                 values  = 1 (True), 0 (False), or None (Missing/NaN)
//
// OUTPUT:
//   predicted_labels            — Predicted supplementary-information labels for the
//                                 target project across all 41 queries

FUNCTION Aggregate_Supplementary_Information(retrieved_projects,supplementary_label_matrix):

    // =========================================================
    // 1. INITIALIZATION
    // =========================================================
    predicted_labels = INITIALIZE_EMPTY_DICTIONARY()

    query_ids = GET_ALL_QUERY_IDS(supplementary_label_matrix)
            // expected query count = 41

    // =========================================================
    // 2. QUERY-WISE VOTING
    // =========================================================
    FOR EACH query_id IN query_ids:

        project_labels = INITIALIZE_EMPTY_LIST()

        FOR EACH project IN retrieved_projects:

            label = supplementary_label_matrix[project][query_id]
                    // label ∈ {1, 0, None}

            project_labels.APPEND(label)

        predicted_label = CALCULATE_MAJORITY_VOTE(project_labels)

        predicted_labels[query_id] = predicted_label

    RETURN predicted_labels

FUNCTION CALCULATE_MAJORITY_VOTE(project_labels):

    /*
    * project_labels:
    *   Labels from the top-k retrieved projects for a single query,
    *   ordered by descending similarity of retrieved projects.
    *
    *   In this study, k = 3.
    *
    *   Valid values:
    *      1    = True  / statement is present
    *      0    = False / statement is explicitly not required or not applicable
    *      None = information is absent or not reported
    */

    // =========================================================
    // 1. TOTAL ABSENCE OF INFORMATION
    // =========================================================
    IF ALL values in project_labels are None THEN
        RETURN None

    // =========================================================
    // 2. HARD VOTING BY MAJORITY AGREEMENT
    // =========================================================
    count_true  = COUNT_VALUES(project_labels, value = 1)
    count_false = COUNT_VALUES(project_labels, value = 0)
    count_none  = COUNT_VALUES(project_labels, value = None)

    IF count_true >= 2 THEN
        RETURN 1

    ELSE IF count_false >= 2 THEN
        RETURN 0

    // =========================================================
    // 3. SINGLE VALID OBSERVATION OVERRIDES ABSENT INFORMATION
    // =========================================================
    ELSE IF count_true == 1 AND count_none == 2 THEN
        RETURN 1

    ELSE IF count_false == 1 AND count_none == 2 THEN
        RETURN 0

    // =========================================================
    // 4. SIMILARITY-BASED TIE-BREAKING FOR CONFLICTING LABELS
    // =========================================================
    ELSE
        /*
        * Example conflict patterns:
        *   [1, 0, None]
        *   [0, 1, None]
        *   [None, 1, 0]
        *
        * Since project_labels are ordered by descending similarity,
        * the nearest retrieved project with a valid non-null label
        * is used to break the tie.
        */

        FOR EACH label IN project_labels:

            IF label IS NOT None THEN
                RETURN label

