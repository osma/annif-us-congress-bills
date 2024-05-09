# Annif subject classification for US Congress bills

This repo contains code and documentation for applying
[Annif](https://annif.org) for the subject classification of US Congress
bills. It was inspired by the
[publication](https://www.linkedin.com/posts/ari-hershowitz_dreamproitbilllabelsus-datasets-at-hugging-activity-7193325364230721536-fz61)
of the
[bill_labels_us](https://huggingface.co/datasets/dreamproit/bill_labels_us)
data set on Hugging Face Hub by Ari Hershowitz. It shows how to predict
policy areas (multiclass classification: pick the best one out of 34 policy
areas) as well as legislative subjects (multilabel classification: pick the
most relevant subjects out of 4658 legislative subjects) based only on the
title and text of the bill.

## Data preparation

The Jupyter notebook [dataset-to-corpus.ipynb)(dataset-to-corpus.ipynb)
loads the original data set directly from Hugging Face Hub using the
`datasets` library and then converts it into corpus files in a format
suitable for Annif. The original data set is also split into 90% train /
10% test subsets. The notebook will produce the following files:

- [policy_area.csv](policy_area.csv): the 34 policy areas as an Annif vocabulary in CSV format
- policy_area-test.tsv.gz: train set for the policy area prediction task (107k examples)
- policy_area-train.tsv.gz: test set for the policy area prediction task (12k examples)
- [legislative_subjects.csv](legislative_subjects.csv): the 4658 legislative subjects as an Annif vocabulary in CSV format
- legislative_subject-test.tsv.gz: train set for the legislative subjects prediction task (107k examples)
- legislative_subject-train.tsv.gz: test set for the legislative prediction task (12k examples)

(The train and test sets are not included in this repository because they
take up around 670MB, which is too big for plain git without LFS.)

## Annif projects

The Annif configuration file [projects.cfg](projects.cfg) defines six
projects, three for each prediction task:

- `pa-tfidf-en`: TFIDF prediction of policy areas
- `pa-parabel-en`: Omikuji Parabel prediction of policy areas
- `pa-svc-en`: Support Vector Classification prediction of policy areas
- `ls-mllm-en`: MLLM (lexical) prediction of legislative subjects
- `ls-parabel-en`: Omikuji Parabel prediction of legislative subjects
- `ls-bonsai-en`: Omikuji Bonsai prediction of legislative subjects (using ngram=2 and min_df=3 settings for improved accuracy)

Snowball stemming is used in all six projects. Minimum token size is 3
characters, which is the default for Annif.

## Training Annif

To learn how to install and use Annif, see the [Annif GitHub repository](https://github.com/NatLibFi/Annif)
and the [Annif tutorial](https://github.com/NatLibFi/Annif-tutorial). 

Here are the commands used to train the models:

```bash
# load the policy area vocabulary
annif load-vocab pa policy_area.csv
# train the policy area models
annif train pa-tfidf-en policy_area-train.tsv.gz
annif train pa-parabel-en policy_area-train.tsv.gz
annif train pa-svc-en policy_area-train.tsv.gz

# load the legislative subjects vocabulary
annif load-vocab ls legislative_subjects.csv
# train the legislative subjects models
annif train ls-mllm-en legislative_subject-train.tsv.gz
annif train ls-parabel-en legislative_subject-train.tsv.gz
annif train ls-bonsai-en legislative_subject-train.tsv.gz
```

## Evaluation

The projects were evaluated using the `annif eval` command against the test
set records. The legislative subject projects were additionally evaluated
using the `annif optimize` command to find out the `limit` and `threshold`
parameters that maximize the F1 score. The precision and recall have been
reported using the same values for `limit` and `threshold`.

## Results

### Policy areas

For the policy area prediction, the results were as follows:

| Project | Accuracy |   nDCG | Model size |
|---------|---------:|-------:|-----------:| 
| TFIDF   |   0.5381 | 0.7129 |     5.1 MB |
| Parabel |   0.9001 | 0.9531 |     8.8 MB |
| SVC     |   0.9011 | 0.9536 |      12 MB |

Findings:

- The baseline TFIDF method performed quite poorly.
- Parabel and SVC gave very similar results with more than 90% accuracy. It seems likely that the remaining 10% were difficult cases that may be impossible to automatically classify based on the title and text alone.
- All models are very small mainly due to the small vocabulary

### Legislative subjects

For the legislative subjects prediction, the results were as follows:

| Project | limit | threshold | precision | recall |     F1 | Model size |
|---------|------:|----------:|----------:|-------:|-------:|-----------:|
| MLLM    |    15 |      0.15 |    0.3834 | 0.1897 | 0.2210 |       1 MB |
| Parabel |    15 |      0.15 |    0.6983 | 0.7072 | 0.6717 |     174 MB |
| Bonsai  |    15 |      0.15 |    0.7762 | 0.7540 | 0.7372 |     597 MB |

Findings:

- The MLLM lexical method performs poorly, probably because the vocabulary is just a flat list of labels without any synonyms or semantic relations.
- Parabel and Bonsai both give excellent results. Bonsai, which uses 2-grams, achieves a F1 score of almost 0.74, which is so good it is almost suspicious.
- Model sizes are relatively modest, although the ngram=2 setting clearly increases the size of the Bonsai model compared to the otherwise similar Parabel model.

## Author

Osma Suominen

## License

[CC0](LICENSE)
