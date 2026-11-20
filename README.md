# Mentalization RAG migration bundle

This folder consolidates the mentalization/adolescent-support RAG assets found in `teen_mh_agent` and their source books.

## Contents

- `knowledge/`: legacy FAISS index and metadata.
- `knowledge_v2/`: SQLite knowledge stores plus FAISS/BM25/chunk indexes used by the current hybrid retrieval pipeline.
- `knowledge_lightrag/`: LightRAG graph and vector stores (`mentalization_guidance_v2` and `mentalization_guidance_v2_bge_m3`).
- `source_materials/books/`: the three book PDFs listed as required primary sources in the response-guidance source manifest.
- `source_materials/备用书籍/`: other book files from `references/books/` that were not listed as sources for this RAG; kept separately for future use.
- `source_materials/provenance/response_guidance_v2/`: source manifest, pipeline notes, extracted evidence atoms, guidance cards, verification/curration outputs, and build reports.

## Book sources

1. *Mentalization-Based Treatment for Personality Disorders: A Practical Guide* — Anthony Bateman and Peter Fonagy, 2nd edition (2016).
2. *心智化：依附关系、情感调节、自我发展* — local Chinese translation of the foundational work by Peter Fonagy, György Gergely, Elliot Jurist, and Mary Target (2002).
3. *Mentalization-Based Treatment for Adolescents: A Practical Treatment Guide* — Trudie Rossouw, Maria Wiwe, and Ioanna Vrouva, 1st edition (2021), DOI: 10.4324/9780429323928.

## Notes

The index manifests retain their original build settings and source paths. Those paths refer to the original workspace and may need updating if this bundle is moved. Embedding indexes may also require the same embedding model/runtime used at build time; see each `teen_mh_manifest.json` and the source pipeline documentation. This bundle is a file-level consolidation, not a rebuild or retest of the indexes.
