# Clickbait Spoiler Generation

## Project Goal

This project generates spoilers for clickbait contents. In this context, a spoiler is a short answer that reveals the hidden information behind a clickbait headline, so the reader does not need to open the article to satisfy the curiosity gap.

We use the Webis Clickbait Spoiling Corpus 2022 and compare three simple LLM prompting strategies.

The project follows the SemEval-2023 clickbait spoiling task setup [[1]](https://doi.org/10.18653/v1/2023.semeval-1.312). Our main idea is to generate short spoilers that answer the curiosity gap in the headline while staying grounded in the article text.

## Dataset

We use the Webis Clickbait Spoiling Corpus 2022 from Zenodo [[2]](https://doi.org/10.5281/zenodo.8136637). The dataset has predefined train, validation, and test splits:

| Split | Rows |
|---|---:|
| Train | 3,200 |
| Validation | 800 |
| Test | 1,000 |

The main fields used in our notebooks are `postText`, `targetParagraphs`, `spoiler`, and `tags`. The spoiler type is stored as `phrase`, `passage`, or `multi`. In the prompts, `multi` is treated as multipart.

## Methodology

The generation notebook first loads the dataset and joins list-based text fields into normal strings. We shorten the article context to 3,000 characters before sending it to the model.

For each selected row, we save:

| Column | Meaning |
|---|---|
| `first_sentence_baseline` | first sentence of the article context |
| `direct_llama_spoiler` | spoiler generated directly from headline and article context |
| `question_llama_spoiler` | spoiler generated after converting the headline into a question |
| `type_aware_llama_spoiler` | question-based spoiler with the gold spoiler type added to the prompt |

The output files from generation are saved in the `output` folder as CSV and JSONL.

## Ollama LLM Spoiler Generation

We use `llama3:latest` through the WU Ollama-compatible endpoint. The notebook calls the `/api/generate` endpoint with non-streaming requests. The user has to be connected to the WU VPN to run the LLM calls.

The prompts tell the model to use only the article context, avoid invented facts, and return only the spoiler. The type-aware prompt adds a short formatting rule:

| Spoiler type | Prompt rule |
|---|---|
| phrase | short phrase |
| passage | one short sentence |
| multipart | short semicolon-separated list |

The current notebook uses `RUN_LIMIT` to control how many examples are processed. In the inspected notebook, `RUN_LIMIT = 10` is set for a small run, but the saved project outputs were produced from a larger run.

## Question Conversion

The question conversion step rewrites the clickbait headline into one short question. This follows the idea that spoiler generation can be helped by explicitly modeling the question behind the clickbait headline [[3]](https://doi.org/10.18653/v1/2023.inlg-main.32).

Example workflow:

1. Convert headline to question.
2. Generate a spoiler from the question and article context.
3. Optionally add spoiler type information.

## BERTScore Evaluation

The evaluation notebook compares generated spoilers with the gold spoilers using BERTScore. This follows related clickbait spoiler work that uses semantic similarity metrics for automatic evaluation [[3]](https://doi.org/10.18653/v1/2023.inlg-main.32), [[4]](https://doi.org/10.1007/s42001-024-00252-z), [[5]](https://doi.org/10.18653/v1/2023.semeval-1.238).

The notebook uses:

| Setting | Value |
|---|---|
| package | `bert-score` |
| language | `lang="en"` |
| baseline scaling | `rescale_with_baseline=True` |
| reference | `gold_spoiler` |

BERTScore measures semantic similarity using contextual embeddings from a transformer model. It is useful, but it was not sufficient to evaluate LLM output on its own. Because of that, we also used manual evaluation.

## Manual Evaluation

Manual evaluation is stored in `output/spoiler_generation_results_manual_eval.xlsx`. The workbook contains 295 generated rows. Of these, 45 rows have complete manual scores.

Groundedness checks whether the spoiler is supported by the article text. Usefulness checks whether the spoiler answers the clickbait and removes the curiosity gap.

We scored each method using:

| Score | Meaning |
|---:|---|
| 0 | not supported, hallucinated, or not useful |
| 0.5 | partly supported or partly useful |
| 1 | fully supported or fully useful |

The manual columns are groundedness and usefulness for each method.

## Main Results

The following results come from `spoiler_generation_results_manual_eval.xlsx`.

### BERTScore Averages

| Method | Precision | Recall | F1 |
|---|---:|---:|---:|
| Direct | 0.2197 | 0.3639 | 0.2882 |
| Question | 0.3920 | 0.3241 | 0.3533 |
| Type-aware | 0.3722 | 0.3574 | 0.3633 |

Type-aware generation has the highest BERTScore F1 in the Excel results. Question-based generation is close, while direct generation has the lowest F1.

### Manual Evaluation Averages

| Method | Groundedness | Usefulness | Average manual score |
|---|---:|---:|---:|
| Direct | 0.9667 | 0.9222 | 0.9444 |
| Question | 0.9000 | 0.7000 | 0.8000 |
| Type-aware | 0.9000 | 0.7556 | 0.8278 |

The manual scores show a different picture from BERTScore. Direct generation is strongest in manual groundedness and usefulness, while type-aware generation is strongest by BERTScore F1.

## Limitations

This is a prompt-based generation project, not a fine-tuned spoiler model. The article context is truncated, so the answer can be missing if it appears later in the article. Multipart spoilers are harder because they often require several facts. BERTScore is useful for semantic similarity, but it does not fully check grounding or hallucination. The manual evaluation is also small, with 45 scored rows in the Excel workbook.

The LLM output depends on the WU Ollama endpoint, network access, and the selected run limit.

## How to Run

1. Make sure the dataset files are in `data/`.
2. Open `clickbait_spoiler_generation.ipynb`.
3. Check the WU network or VPN connection for the Ollama endpoint.
4. Set `RUN_LIMIT` to a small number for testing, or increase it for a larger run.
5. Run the generation notebook to create `output/spoiler_generation_results.csv` and `output/spoiler_generation_results.jsonl`.
6. Open `clickbait_spoiler_eval.ipynb`.
7. Run the BERTScore evaluation to create the scored output files.
8. Use `output/spoiler_generation_results_manual_eval.xlsx` for the manual evaluation summary.

## References

[1] M. Fröbe, B. Stein, T. Gollub, M. Hagen, and M. Potthast, "SemEval-2023 Task 5: Clickbait Spoiling," in *Proceedings of the 17th International Workshop on Semantic Evaluation (SemEval-2023)*, 2023, pp. 2275-2286. [Online]. Available: [https://doi.org/10.18653/v1/2023.semeval-1.312](https://doi.org/10.18653/v1/2023.semeval-1.312)

[2] M. Hagen, M. Fröbe, A. Jurk, and M. Potthast, *Webis Clickbait Spoiling Corpus 2022 (1.0.0)*. Zenodo, 2022. [Online]. Available: [https://doi.org/10.5281/zenodo.8136637](https://doi.org/10.5281/zenodo.8136637)

[3] M. Woźny and M. Lango, "Generating clickbait spoilers with an ensemble of large language models," in *Proceedings of the 16th International Natural Language Generation Conference*, 2023, pp. 431-436. [Online]. Available: [https://doi.org/10.18653/v1/2023.inlg-main.32](https://doi.org/10.18653/v1/2023.inlg-main.32)

[4] I. Panda, J. P. Singh, G. Pradhan, and K. Kumari, "A deep learning framework for clickbait spoiler generation and type identification," *Journal of Computational Social Science*, vol. 7, no. 1, pp. 671-693, 2024. [Online]. Available: [https://doi.org/10.1007/s42001-024-00252-z](https://doi.org/10.1007/s42001-024-00252-z)

[5] J. Keller, N. Rehbach, and I. Zafar, "nancy-hicks-gribble at SemEval-2023 Task 5: Classifying and generating clickbait spoilers with RoBERTa," in *Proceedings of the 17th International Workshop on Semantic Evaluation (SemEval-2023)*, 2023, pp. 1712-1717. [Online]. Available: [https://doi.org/10.18653/v1/2023.semeval-1.238](https://doi.org/10.18653/v1/2023.semeval-1.238)
