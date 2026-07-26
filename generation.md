// INPUT:
//   verified_context        — Output of HITL_Verify_Context (see hitl-verification.md):
//                              predictions_as_text (verified answers to the 41 supplementary
//                              queries), manual_inputs (free-text answers to the optional
//                              concession-specific questions), and top_projects (top 3
//                              retrieved projects, rank-ordered by similarity, each with its
//                              structured concession metadata)
//   curated_project_summary — Natural-language serialization of the current project's
//                              24 concept-guided feature values, derived from its scrutiny
//                              report (see feature-engineering.md), e.g.
//                              "Basic Permissible FSI: 1.33"
//   llm_model                — GPT-4o
//
// OUTPUT: generated_concession_justification — Markdown-formatted concession report text,
//         presented to the user as the final drafted output

FUNCTION Generate_Concession_Justification(verified_context, curated_project_summary, llm_model):

    // =========================================================
    // 1. SYSTEM-LEVEL INSTRUCTION PROMPT
    // =========================================================
    system_prompt = BUILD_SYSTEM_PROMPT(
        role = "Assistant to the Architect, drafting an Open Space Deficiency concession report",
        sections = ['Concession Required', 'Provisions of DCPR', 'Approval Required From',
                    'Justification by Architect'],
        justification_subsections = ['Hardship', 'Fire Safety', 'Structural Safety',
                                      'Public and Health Safety', 'Neighborhood Safety'],
        formatting_rules = [
            'write in detail and bullet points, especially under Hardship',
            'use Markdown tables where they improve readability',
            'do not include a separate Comments-by-AE section; draw inspiration from AE ' +
                'comments and merge relevant points into the Justification subsections instead',
            'no summary section at the end'
        ]
    )

    // =========================================================
    // 2. USER PROMPT ASSEMBLY (ordered)
    // =========================================================
    // Fixed sequence: verified context first, followed by project-specific data,
    // and archival exemplars last — prioritizing validated context over
    // unfiltered precedent reuse.

    user_prompt = ""

    // --- (1) User-verified supplementary information ---
    user_prompt += DELIMIT("~~~", verified_context.predictions_as_text)
                    // question-answer pairs for the 41 queries, reflecting voting
                    // predictions and any HITL overrides (see hitl-verification.md)

    // --- (2) Optional additional contextual information ---
    optional_qa_text = ""
    FOR EACH question, answer IN verified_context.manual_inputs:
        optional_qa_text += f"Question: {question}\nAnswer: {answer}\n"
    user_prompt += DELIMIT("$$$", optional_qa_text)

    // --- (3) Key feature values from the current project's scrutiny report ---
    user_prompt += DELIMIT("###", curated_project_summary)

    // --- (4) Structured concession reports of the top 3 retrieved projects ---
    sample_reports_text = ""
    FOR EACH rank, project IN ENUMERATE(verified_context.top_projects):
        sample_reports_text += f"\n**Project example {rank + 1}:**\n{project.concession_data.text}\n"
    user_prompt += DELIMIT("```", sample_reports_text)

    // =========================================================
    // 3. LLM GENERATION
    // =========================================================
    // Default API parameters are used (temperature = 1.0, top_p = 1.0)
    generated_concession_justification = LLM_GENERATE(
        model = llm_model,
        system_prompt = system_prompt,
        user_prompt = user_prompt,
        temperature = 1.0,
        top_p = 1.0
    )

    // =========================================================
    // 4. RETURN GENERATED JUSTIFICATION
    // =========================================================
    RETURN generated_concession_justification
