// INPUT:
//   concession_reports    — Set of normalized concession report texts, one per archival
//                            project (PDF converted to plain text, with OCR-preserved
//                            table structure; see Handling of Unstructured Concession
//                            Reports). Processed in chronological submission order.
//   scrutiny_reports      — Corresponding set of HTML scrutiny reports, used to verify
//                            whether a candidate concept's value is already explicitly
//                            present in the input drawing/model.
//   dcpr_bounds           — Lookup table of DCPR-prescribed regulatory limits (as in
//                            feature-engineering.md), used to verify whether a candidate
//                            concept's value is directly derivable from code.
//   min_occurrence_count  — Minimum number of projects in which a candidate concept must
//                            recur to be treated as generalizable rather than case-specific
//
// OUTPUT: Concept pool of 50 retained concepts (of 69 disambiguated concepts identified),
//         split into feature_concepts (24, routed to feature-engineering.md) and
//         supplementary_concepts (26, routed to supplementary-information query design),
//         saved as a structured concept table

FUNCTION Extract_And_Classify_Concepts(concession_reports, scrutiny_reports, dcpr_bounds,
                                        min_occurrence_count):

    // =========================================================
    // 1. CONCEPT EXTRACTION (per report, LLM-based)
    // =========================================================
    // Following the authors' earlier knowledge-graph construction work, an LLM is
    // prompted with zero-shot chain-of-thought instructions to identify concept
    // mentions in each concession report, spanning objects, locations, entities,
    // roles of persons, documents, services, and conditions. Functional roles
    // (e.g., "licensed structural engineer") are explicitly prioritized over named
    // individuals or organizations to preserve generalizability across projects.

    raw_concepts_by_report = INITIALIZE_EMPTY_MAP()

    FOR EACH report IN concession_reports (IN SUBMISSION ORDER):
        candidate_concepts = LLM_EXTRACT_CONCEPTS(report, prompt=ZERO_SHOT_COT_CONCEPT_PROMPT)
        // candidate_concepts: list of (surface_form, concept_type) pairs
        // concept_type IN {object, location, entity, role, document, service, condition}
        raw_concepts_by_report[report.id] = candidate_concepts

    // =========================================================
    // 2. ENTITY DISAMBIGUATION
    // =========================================================
    // Merge duplicate or synonymous surface forms referring to the same underlying
    // concept (e.g., "structural engineer" vs. "licensed structural engineer") into
    // a single canonical concept name, resolved against a running registry so that
    // later reports map onto concepts already identified in earlier reports.

    concept_registry = INITIALIZE_EMPTY_SET()
    canonical_concepts_by_report = INITIALIZE_EMPTY_MAP()

    FOR EACH report_id, candidate_concepts IN raw_concepts_by_report:
        canonical_names = DISAMBIGUATE(candidate_concepts, concept_registry)
        concept_registry = concept_registry UNION canonical_names
        canonical_concepts_by_report[report_id] = canonical_names

    // =========================================================
    // 3. SATURATION TRACKING (concept accumulation across reports)
    // =========================================================
    // Track the cumulative set of unique concepts as reports are processed in
    // submission order, recording how many concepts are newly introduced by each
    // report. Extraction continues until the marginal growth in unique concepts
    // approaches zero across successive reports (vocabulary saturation).

    seen_concepts = INITIALIZE_EMPTY_SET()
    saturation_curve = INITIALIZE_EMPTY_LIST()

    FOR EACH report_id, concepts IN canonical_concepts_by_report (IN SUBMISSION ORDER):
        new_concepts = concepts - seen_concepts
        seen_concepts = seen_concepts UNION concepts

        saturation_curve.APPEND({
            'report_id':            report_id,
            'new_concepts_added':   LEN(new_concepts),
            'total_unique_concepts': LEN(seen_concepts)
        })

    ASSERT IS_SATURATED(saturation_curve)
    // e.g., new_concepts_added falls to ~0 over the final reports processed

    concept_pool = seen_concepts   // 69 disambiguated concepts

    // =========================================================
    // 4. CLASSIFICATION BY DATA TYPE
    // =========================================================
    // Each concept is assigned to exactly one of five categories, based on how its
    // value can be obtained and its intended role in retrieval and augmentation.
    classified_concepts = INITIALIZE_EMPTY_MAP()

    FOR EACH concept IN concept_pool:

        occurrence_count = COUNT_PROJECTS_CONTAINING(concept, canonical_concepts_by_report)

        // --- Case-specific data ---
        // Concepts recurring in only one or two projects are too context-dependent
        // to generalize into features or supplementary-information queries
        // (e.g., "training of drainage", "error in survey")
        IF occurrence_count <= min_occurrence_count:
            category = "case_specific_data"

        // --- Data available in the input drawing/model ---
        // Explicitly present in the HTML scrutiny report (geometric/geospatial values)
        ELSE IF IS_DIRECTLY_PRESENT_IN_SCRUTINY_REPORT(concept, scrutiny_reports):
            category = "data_in_drawing_model"

        // --- Code requirements ---
        // Obtainable directly from DCPR based on project design parameters
        // (e.g., "maximum permissible FSI", "minimum required open space")
        ELSE IF IS_DERIVABLE_FROM_DCPR(concept, dcpr_bounds):
            category = "code_requirement"

        // --- Derived data ---
        // Higher-level interpretive outcomes inferred from multiple interrelated
        // factors rather than directly observable/verifiable inputs
        // (e.g., fire safety, hardship, structural safety, health of inhabitants)
        ELSE IF IS_INTERPRETIVE_OUTCOME(concept):
            category = "derived_data"

        // --- Supplementary information ---
        // Articulated by architects in concession reports but not available in the
        // submitted drawing/model or derivable from code
        ELSE:
            category = "supplementary_information"

        classified_concepts[concept] = category

    // =========================================================
    // 5. PRUNING
    // =========================================================
    // Exclude concepts representing interpretive outcomes (would encode conclusions
    // rather than underlying project conditions) or insufficiently generalizable,
    // context-specific occurrences.
    retained_concepts = INITIALIZE_EMPTY_MAP()

    FOR EACH concept, category IN classified_concepts:
        IF category == "derived_data" OR category == "case_specific_data":
            CONTINUE   // excluded from downstream feature/supplementary-info construction
        retained_concepts[concept] = category

    // retained_concepts now holds 50 concepts (of the original 69)

    // =========================================================
    // 6. ROUTE CONCEPTS TO DOWNSTREAM CONSTRUCTION
    // =========================================================
    feature_concepts       = INITIALIZE_EMPTY_LIST()   // -> feature-engineering.md
    supplementary_concepts = INITIALIZE_EMPTY_LIST()   // -> supplementary-information queries

    FOR EACH concept, category IN retained_concepts:
        IF category == "data_in_drawing_model" OR category == "code_requirement":
            feature_concepts.APPEND(concept)           // 24 concepts
        ELSE IF category == "supplementary_information":
            supplementary_concepts.APPEND(concept)      // 26 concepts

    // =========================================================
    // 7. SAVE CONCEPT POOL
    // =========================================================
    concept_pool_table = BUILD_TABLE(retained_concepts, saturation_curve)
                          // columns: Name, Category, Source_Reports

    SAVE_AS_EXCEL(concept_pool_table, "concept_pool.xlsx")

    RETURN feature_concepts, supplementary_concepts
