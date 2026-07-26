// INPUT:
//   predicted_supplementary_info — Output of Aggregate_Supplementary_Information (see
//                                   voting.md): one predicted {1 (True), 0 (False), None}
//                                   value per each of the 41 queries
//   query_bank                    — 41 supplementary-information queries (see
//                                   supplementary-information-prediction.md), used to render
//                                   each prediction with its human-readable question text
//   top_projects                  — Output of Retrieve_Similar_Projects (see retrieval.md):
//                                   top 3 archival projects, rank-ordered by similarity, each
//                                   with its structured concession metadata
//   optional_query_bank           — Free-text clarifying questions for the selected
//                                   concession type (e.g., 3 questions for Open Space
//                                   Deficiency), allowing the architect to supply
//                                   project-specific context not captured by the 41
//                                   structured queries
//
// OUTPUT: verified_context — user-reviewed/edited supplementary-information values (as
//         formatted text), user-provided free-text answers, and the retrieved projects,
//         consolidated and passed forward to generation (see generation.md)

FUNCTION HITL_Verify_Context(predicted_supplementary_info, query_bank, top_projects,
                              optional_query_bank):

    // =========================================================
    // 1. DISPLAY PREDICTED SUPPLEMENTARY INFORMATION FOR REVIEW
    // =========================================================
    // Each of the 41 predicted values is presented for review, pre-filled with the
    // voted prediction (True, False, or None), allowing the user to review and, if
    // necessary, modify the value based on project-specific context.

    selected_values = INITIALIZE_EMPTY_MAP()

    FOR EACH query_id, predicted_value IN predicted_supplementary_info:
        selected_values[query_id] = RENDER_TRISTATE_INPUT(label = query_bank[query_id].text,
                                                            default = predicted_value,
                                                            options = [True, False, None])
        // user may accept the prediction or override it; final state captured on submission

    // =========================================================
    // 2. CAPTURE OPTIONAL USER-PROVIDED INPUTS
    // =========================================================
    // Free-text questions specific to the selected concession type, allowing the
    // user to supply clarifications or edge cases not captured by the 41
    // structured queries.

    manual_inputs = INITIALIZE_EMPTY_MAP()

    FOR EACH question IN optional_query_bank:
        manual_inputs[question] = RENDER_TEXT_INPUT(label = question)
        // left blank if the user has no additional input for this question

    // =========================================================
    // 3. DISPLAY RETRIEVED PROJECTS
    // =========================================================
    // The top 3 retrieved projects are made available to the user, so they may be
    // consulted directly as reference precedents for the retrieval decision.
    DISPLAY(top_projects)

    // =========================================================
    // 4. CONSOLIDATE VERIFIED CONTEXT
    // =========================================================
    // Serialize the (possibly overridden) supplementary-information selections
    // into formatted text, and combine with the manually-entered answers. This
    // consolidated, user-verified context is what gets passed forward.

    ON SUBMIT("Generate Concession"):
        verified_predictions_text = ""
        FOR EACH query_id, value IN selected_values:
            verified_predictions_text += f"{query_bank[query_id].text} {value}\n"

        verified_context = {
            'predictions_as_text': verified_predictions_text,
            'manual_inputs':       manual_inputs,
            'top_projects':        top_projects
        }

        RETURN verified_context
