# CG-RAG-HITL: Pseudocode and Supporting Materials

This repository provides supporting materials for the study:

**Automating Subjective Regulatory Justifications in Building Permit Workflows Using Concept-Guided Retrieval-Augmented Generation (RAG) and Human-in-the-Loop Verification**

The study develops a Concept-Guided Retrieval-Augmented Generation (CG-RAG) framework with Human-in-the-Loop (HITL) verification to support architects in drafting subjective concession justifications in building permit workflows. The framework was evaluated using Open Space Deficiency (OSD) concession cases from Mumbai, India.

## Repository Purpose

The complete implementation code cannot be publicly released because parts of the developed system are proprietary.

To support methodological transparency and reproducibility, this repository provides pseudocode for the principal development- and application-stage components of the proposed framework. The materials cover concept identification, structured feature construction, supplementary-information extraction, precedent retrieval, voting-based prediction, human verification, and LLM-assisted generation.

## Framework Overview

The pseudocode is organized into two stages:

1. **Development stage:** process archival concession and scrutiny reports to identify reusable concepts, construct the feature-vector database, and extract supplementary-information labels.
2. **Application stage:** process a new project, retrieve similar archival projects, predict supplementary information, obtain human verification, and generate a draft concession justification.

The overall sequence is:

```text
Archival concession and scrutiny reports
                |
                v
Concept extraction and classification
                |
                +-------------------------------+
                |                               |
                v                               v
Feature engineering                 Supplementary-information extraction
                |                               |
                v                               v
       Feature-vector database          Supplementary-information store
                |                               |
                +---------------+---------------+
                                |
                                v
                Similar-project retrieval
                                |
                                v
                   Voting-based prediction
                                |
                                v
                  Human-in-the-loop review
                                |
                                v
                 Concession-justification generation
```

## Repository Contents

| File | Purpose |
| --- | --- |
| [`pipeline-overview.md`](pipeline-overview.md) | Connects the individual components into the complete development- and application-stage CG-RAG-HITL workflow. |
| [`concept-extraction.md`](concept-extraction.md) | Extracts and disambiguates concepts from archival concession reports, tracks concept saturation, classifies concepts by data source and role, and retains the concepts used downstream. |
| [`feature-engineering.md`](feature-engineering.md) | Extracts structured project attributes from scrutiny reports; derives, imputes, encodes, and scales 49 features representing 24 concepts; and saves the processed feature matrix and preprocessing objects. |
| [`supplementary-information-prediction.md`](supplementary-information-prediction.md) | Uses 41 queries linked to 26 supplementary-information concepts to extract `True`, `False`, or missing labels from archival concession reports. |
| [`retrieval.md`](retrieval.md) | Constructs a schema-aligned query vector for a new project, calculates cosine similarity, retrieves the top three archival projects, and attaches their structured concession metadata. |
| [`voting.md`](voting.md) | Aggregates the 41 supplementary-information labels from the top three retrieved projects using majority voting and similarity-ordered tie-breaking. |
| [`hitl-verification.md`](hitl-verification.md) | Presents predicted values and retrieved precedents for human review, captures overrides and optional project-specific inputs, and consolidates the verified context. |
| [`generation.md`](generation.md) | Combines the verified context, the new project's concept-guided feature summary, and the retrieved precedents to generate a Markdown-formatted concession justification using an LLM. |

## Suggested Reading Order

For an end-to-end understanding of the framework, begin with [`pipeline-overview.md`](pipeline-overview.md). The component-level files can then be read in the following order:

1. [`concept-extraction.md`](concept-extraction.md)
2. [`feature-engineering.md`](feature-engineering.md)
3. [`supplementary-information-prediction.md`](supplementary-information-prediction.md)
4. [`retrieval.md`](retrieval.md)
5. [`voting.md`](voting.md)
6. [`hitl-verification.md`](hitl-verification.md)
7. [`generation.md`](generation.md)

## Dataset

The anonymized data associated with the study, including scrutiny reports and public-domain concession reports for the 333 projects used in the paper, are available through [Mendeley Data](https://data.mendeley.com/datasets/fbytrnrdcr/1).

The dataset also includes information on the publicly accessible concession reports and their original sources.

## Important Notes

- The files in this repository are pseudocode and are not intended to run directly as executable software.
- Function and data-object names express methodological roles and do not expose proprietary implementation details.
- The pseudocode should be read together with the corresponding manuscript for experimental settings, evaluation procedures, and complete methodological context.

## Citation

If you use this repository or the associated dataset, please cite the corresponding paper:

Karmakar, Ankan and Chhetiya, Inder and Delhi, Venkata Santosh Kumar, *Automating Subjective Regulatory Justifications in Building Permit Workflows Using Concept-Guided Retrieval-Augmented Generation (RAG) and Human-in-the-Loop Verification*. Available at SSRN: <https://doi.org/10.2139/ssrn.6388872>

## Contact

For academic queries related to the methodology or supporting materials, please contact the corresponding author listed in the manuscript.
