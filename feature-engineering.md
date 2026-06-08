// INPUT:
//   html_reports   — Set of HTML scrutiny report files, one per building proposal
//   dcpr_bounds    — Lookup table of DCPR-prescribed regulatory limits, keyed by
//                    feature name and zero or more contextual parameters
//                    (e.g., building_height, plot_use, or other plot attributes);
//                    provides permissible min/max values for continuous features
//                    used during scaling and fallback derivation.
//                    The required lookup key varies per feature — some features
//                    have fixed regulatory bounds, others depend on plot context.
//
// OUTPUT: Processed Feature Matrix, saved as pickle

FUNCTION Process_Scrutiny_Reports(html_reports, dcpr_bounds):

    // =========================================================
    // 1. HTML EXTRACTION
    // =========================================================
    feature_matrix = INITIALIZE_EMPTY_DATAFRAME()

    FOR EACH report IN html_reports:
        row = INITIALIZE_EMPTY_ROW()

        // Extract scalar fields
        row['Basic_FSI']       = EXTRACT_VALUE(report, "Basic FSI")
        row['Building_Height'] = EXTRACT_VALUE(report, "Building Height")

        // Extract dimensional setback values from the deficiency table
        deficiency_table       = EXTRACT_TABLE(report, "Deficiency Table")
        row['Front_Required']  = deficiency_table['Front_Required']
        row['Front_Proposed']  = deficiency_table['Front_Proposed']
        row['Rear_Required']   = deficiency_table['Rear_Required']
        row['Rear_Proposed']   = deficiency_table['Rear_Proposed']
        // ... extract remaining fields from deficiency_table

        feature_matrix.APPEND(row)

    // =========================================================
    // 2. FEATURE CREATION (Concept Derivation)
    // =========================================================
    FOR EACH row IN feature_matrix:

        // --- Front Open Space Ratio ---
        // Use DCPR regulatory minimum if the scrutiny report omits it
        IF NOT IS_MISSING(row['Front_Required']):
            front_req = row['Front_Required']
        ELSE:
            front_req = GET_DCPR_VALUE("Front_Open_Space",
                                        row['Building_Height'],
                                        row['Plot_Use'])

        row['Front_Deficiency_Proposed_Pct'] = row['Front_Proposed'] / front_req

        // --- Rear Open Space Ratio ---
        IF NOT IS_MISSING(row['Rear_Required']):
            rear_req = row['Rear_Required']
        ELSE:
            rear_req = GET_DCPR_VALUE("Rear_Open_Space",
                                       row['Building_Height'],
                                       row['Plot_Use'])

        row['Rear_Deficiency_Proposed_Pct'] = row['Rear_Proposed'] / rear_req

        // --- Regulation-Triggering Boolean Conditions ---
        row['CFO_Requirement'] = IF row['Building_Height'] > 32 THEN 1 ELSE 0

        // ... derive remaining features to reach 49 total across 24 concepts

    // =========================================================
    // 3. IMPUTATION
    // =========================================================
    FOR EACH col IN feature_matrix.columns:
        IF HAS_MISSING_VALUES(col):

            IF IS_CONTINUOUS(col):
                // e.g., missing dimensional parameters
                APPLY_KNN_IMPUTER(feature_matrix[continuous_columns_with_missing], k=3)

            ELSE IF IS_BOOLEAN(col) OR IMPLIES_NON_APPLICABILITY(col):
                // e.g., podium-related deficiency where no podium exists
                FILL_MISSING(col, value=0)

            ELSE IF IS_CATEGORICAL(col):
                // Categorical imputation is skipped as no categorical column had missing values
                PASS

    // =========================================================
    // 4. CATEGORICAL ENCODING
    // =========================================================
    // e.g., Parking type: {normal, pit, stacked, rotary}
    categorical_columns = ['Parking_Type', ...]
    FOR EACH col IN categorical_columns:
        feature_matrix = APPLY_ONE_HOT_ENCODING(feature_matrix, col)
        //Fit encoder during development and reuse fitted encoder during application.

    // =========================================================
    // 5. FEATURE SCALING
    // =========================================================
    continuous_columns = ['Basic_FSI', ...]
    FOR EACH col IN continuous_columns:

        // Use DCPR regulatory bounds instead of dataset min/max where defined,
        // to account for valid limit values absent from the training data.
        IF IS_RATIO_BASED(col):
            X_min = 0.0
            X_max = 1.0

        ELSE IF DCPR_BOUNDS_DEFINED(col):
            X_min = dcpr_bounds.GET_MIN(col)
            X_max = dcpr_bounds.GET_MAX(col)
            // Lookup key varies by feature; may depend on building_height,
            // plot_use, other plot attributes, or none (fixed regulatory limit)

        ELSE:
            X_min = MIN(feature_matrix[col])
            X_max = MAX(feature_matrix[col])

        feature_matrix[col] = APPLY_MIN_MAX_SCALING(feature_matrix[col], X_min, X_max)

    // =========================================================
    // 6. SAVE DATA
    // =========================================================
    // Retain only the columns required for CG-RAG:
    // (i) raw features carried forward, (ii) derived features,
    // (iii) one-hot encoded columns; drop intermediate and source columns.

    output_columns = raw_features       // e.g., Basic_FSI, Building_Height, ...
                   + derived_features   // e.g., Front_Deficiency_Proposed_Pct, CFO_Requirement, ...
                   + encoded_columns    // e.g., Parking_Type...

    output_data = feature_matrix.SELECT_COLUMNS(output_columns)
                                        // excludes original categorical columns,
                                        // intermediate ratio denominators, etc.

    SAVE_AS_PICKLE(output_data, "feature_matrix.pkl")

    // Save additional pipeline encoders for future use during application stage
    
    SAVE_AS_PICKLE(KNN_IMPUTER, "continuous_imputer.pkl")
    SAVE_AS_PICKLE(ONE_HOT_ENCODER, "ohe.pkl")
    SAVE_AS_PICKLE(SCALING_PARAMETERS, "scaling_parameters.pkl")

    RETURN output_data