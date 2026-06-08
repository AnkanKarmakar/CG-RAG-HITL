# CG-RAG-HITL: Pseudocode and Supporting Materials

This repository provides supporting materials for the study:

**Automating Subjective Regulatory Justifications in Building Permit Workflows Using Concept-Guided Retrieval-Augmented Generation (RAG) and Human-in-the-Loop Verification**

The study develops a Concept-Guided Retrieval-Augmented Generation (CG-RAG) framework with Human-in-the-Loop (HITL) verification to support architects in drafting subjective concession justifications in building permit workflows. The framework was evaluated using Open Space Deficiency (OSD) concession cases from Mumbai, India.

## Repository Purpose

The complete implementation code cannot be publicly released because parts of the developed system are proprietary. 

To support methodological transparency and reproducibility, this repository provides pseudocode for the core components of the proposed framework:

1. Feature engineering and scaling for structured project information representation.
2. Voting-based retrieval aggregation for supplementary-information prediction.

## Repository Contents

```text
CG-RAG-HITL/
│
├── feature-engineering.md
│   Pseudocode for extracting, engineering, encoding, imputing, and scaling
│   structured features derived from scrutiny reports.
│
├── voting.md
│   Pseudocode for aggregating supplementary-information labels from
│   top-3 retrieved archival projects using the voting logic adopted in
│   the CG-RAG framework.
│
└── README.md
    Repository description and usage notes.
```

## Pseudocode Files

### `feature-engineering.md`

It covers:

* extraction of relevant structured project attributes;
* derivation of concept-guided numerical and categorical features;
* missing-value handling;
* one-hot encoding of categorical attributes;
* feature scaling using regulatory or data-derived bounds;
* saving of the processed feature matrix and preprocessing objects.

### `voting.md`

It covers:

* query-wise aggregation of supplementary-information labels;
* handling of `True`, `False`, and missing values;
* majority voting across retrieved projects;
* similarity-ordered tie-breaking;
* generation of predicted supplementary-information labels for the target project.

## Dataset

The anonymized data associated with the study, including scrutiny reports and public-domain concession reports for the 333 projects used in the paper, are available through Mendeley Data:

https://data.mendeley.com/datasets/fbytrnrdcr/1

The dataset also includes information on the publicly accessible concession reports and their original sources.

## Important Notes

* The files in this repository are pseudocode and are not intended to run directly as executable software.
* The repository is provided for academic transparency and to support reproducibility of the methodological logic described in the paper.

## Citation

If you use this repository or the associated dataset, please cite the corresponding paper:

Karmakar, Ankan and Chhetiya, Inder and Delhi, Venkata Santosh Kumar, Automating Subjective Regulatory Justifications in Building Permit Workflows Using Concept-Guided Retrieval-Augmented Generation (RAG) and Human-in-the-Loop Verification. Available at SSRN: http://dx.doi.org/10.2139/ssrn.6388872

## Contact

For academic queries related to the methodology or supporting materials, please contact the corresponding author listed in the manuscript.
