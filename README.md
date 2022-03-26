# SeqScore
![Build Status](https://github.com/bltlab/seqscore/actions/workflows/main.yml/badge.svg)
[![Documentation Status](https://readthedocs.org/projects/seqscore/badge/?version=latest)](https://seqscore.readthedocs.io/en/latest/?badge=latest)


SeqScore provides scoring for named entity recognition and other
chunking tasks evaluated over sequence labels.


# Installation

To install the latest official release of SeqScore, run:
`pip install seqscore`

This will install the package and add the command `seqscore` in your
Python environment.


# Usage

## Overview

For a list of commands, run `seqscore --help`:
```
$ seqscore --help
Usage: seqscore [OPTIONS] COMMAND [ARGS]...

Options:
  --help  Show this message and exit.

Commands:
  convert
  count
  repair
  score
  validate
```

## Scoring

The most common application of SeqScore is scoring CoNLL-format NER
predictions. Let's assume you have two files, one containing the
correct labels (annotation) and the other containing the predictions
(system output).

The correct labels are in the file [samples/reference.bio](https://github.com/bltlab/seqscore/blob/tutorial/samples/reference.bio):
```
This O
is O
a O
sentence O
. O

University B-ORG
of I-ORG
Pennsylvania I-ORG
is O
in O
West B-LOC
Philadelphia I-LOC
, O
Pennsylvania B-LOC
. O
```

The predictions are in the file [samples/predicted.bio](https://github.com/bltlab/seqscore/blob/tutorial/samples/predicted.bio):
```
This O
is O
a O
sentence O
. O

University B-ORG
of I-ORG
Pennsylvania I-ORG
is O
in O
West B-LOC
Philadelphia B-LOC
, O
Pennsylvania B-LOC
. O
```

To score the predictions, run:
`seqscore score --labels BIO --reference samples/reference.bio samples/predicted.bio`

the result is in [samples/reference_predicted_score.txt](https://github.com/bltlab/seqscore/blob/tutorial/samples/reference_predicted_score.txt)
```
| Type   |   Precision |   Recall |     F1 |   Reference |   Predicted |   Correct |
|--------|-------------|----------|--------|-------------|-------------|-----------|
| ALL    |       50.00 |    66.67 |  57.14 |           3 |           4 |         2 |
| LOC    |       33.33 |    50.00 |  40.00 |           2 |           3 |         1 |
| ORG    |      100.00 |   100.00 | 100.00 |           1 |           1 |         1 |
```

A few things to note:
* The reference file must be specifed with the `--reference`
  flag.
* The chunk encoding (BIO, BIOES, etc.) must be specified
  using the `--labels` flag.
* Both files need to use the same chunk encoding. If you have
  files that use different chunk encodings, use the `convert` command.
* You can get output in a different format using the `--score-format`
  flag.

The above scoring command will work for files that do not have any
invalid transitions, that is, those that perfectly follow what the
encoding allows. However, consider this BIO-encoded file, which we'll
call [samples/invalid.bio](https://github.com/bltlab/seqscore/blob/tutorial/samples/invalid.bio):

```
This O
is O
a O
sentence O
. O

University I-ORG
of I-ORG
Pennsylvania I-ORG
is O
in O
West B-LOC
Philadelphia I-LOC
, O
Pennsylvania B-LOC
. O
```

Note that the token `University` has the label `I-ORG`, but there is
no preceding `B-ORG`. If we try to score it as before with `seqscore
score --labels BIO --reference samples/reference.bio samples/invalid.bio`, scoring
will fail with an error in the file [samples/reference_invalid_score.txt](https://github.com/bltlab/seqscore/blob/tutorial/samples/reference_invalid_score.txt):
```
seqscore.encoding.EncodingError: Stopping due to validation errors in invalid.bio:
Invalid transition 'O' -> 'I-ORG' for token 'University' on line 7
```

To score output with invalid transitions, we need to specify a repair
method which can correct them. We can tell SeqScore to use the same
approach that conlleval uses (which we refer to as "begin" repair in our
paper): `seqscore score --labels BIO --repair-method conlleval  --reference samples/reference.bio samples/invalid.bio`, the result will be in the file [samples/reference_invalid_repair_score.txt](https://github.com/bltlab/seqscore/blob/tutorial/samples/reference_invalid_repair_score.txt):

```
Validation errors in sequence at line 7 of invalid.bio:
Invalid transition 'O' -> 'I-ORG' for token 'University' on line 7
Used method conlleval to repair:
Old: ('I-ORG', 'I-ORG', 'I-ORG', 'O', 'O', 'B-LOC', 'I-LOC', 'O', 'B-LOC', 'O')
New: ('B-ORG', 'I-ORG', 'I-ORG', 'O', 'O', 'B-LOC', 'I-LOC', 'O', 'B-LOC', 'O')
| Type   |   Precision |   Recall |     F1 |   Reference |   Predicted |   Correct |
|--------|-------------|----------|--------|-------------|-------------|-----------|
| ALL    |      100.00 |   100.00 | 100.00 |           3 |           3 |         3 |
| LOC    |      100.00 |   100.00 | 100.00 |           2 |           2 |         2 |
| ORG    |      100.00 |   100.00 | 100.00 |           1 |           1 |         1 |
```

You can use the `-q` flag to suppress the logging of all of the repairs
applied. You may want to also explore the `discard` repair, which can
produce higher scores for output from models without a CRF or constrained
decoding as they are more likely to produce invalid transitions. For example,
the result of running `seqscore score --labels BIO --repair-method discard --reference samples/reference.bio samples/invalid.bio` is 
in the file [samples/reference_invalid_discard_score.txt](https://github.com/bltlab/seqscore/blob/tutorial/samples/reference_invalid_discard_score.txt).

## Validate

To check if a file has any invalid transitions, we can run `seqscore validate --labels BIO samples/reference.bio` and the result will be in [samples/reference_validate.txt](https://github.com/bltlab/seqscore/blob/tutorial/samples/reference_validate.txt):
 ```
 No errors found in 0 tokens, 2 sequences, and 1 documents in reference.bio
 ```
 
 For the example of the [samples/invalid.bio](https://github.com/bltlab/seqscore/blob/tutorial/samples/invalid.bio), we can run `seqscore validate --labels BIO samples/invalid.bio` and the result is in [samples/invalid_validate.txt](https://github.com/bltlab/seqscore/blob/tutorial/samples/invalid_validate.txt):
 ```
Encountered 1 errors in 1 tokens, 2 sequences, and 1 documents in invalid.bio
Invalid transition 'O' -> 'I-ORG' for token 'University' on line 7
 ```
## Convert

We can convert a file from one encoding to another. For example, we can run `seqscore convert --input-labels BIO --output-labels BIOES samples/reference.bio samples/reference_convert_BIO_BIOES.bio` to convert the [samples/reference.bio](https://github.com/bltlab/seqscore/blob/tutorial/samples/reference.bio) from BIO encoding to BIOES encoding. The result is in [reference_convert_BIO_BIOES.bio](https://github.com/bltlab/seqscore/blob/tutorial/samples/reference_convert_BIO_BIOES.bio):
```
This O
is O
a O
sentence O
. O

University B-ORG
of I-ORG
Pennsylvania E-ORG
is O
in O
West B-LOC
Philadelphia E-LOC
, O
Pennsylvania S-LOC
. O
```

We can get a list of available input and output encoding labels by running `seqscore convert --help`:
```
Usage: seqscore convert [OPTIONS] FILE OUTPUT_FILE

Options:
  --file-encoding TEXT            [default: UTF-8]
  --ignore-comment-lines
  --ignore-document-boundaries / --use-document-boundaries
  --output-delim TEXT             [default: space]
  --input-labels [BIO|BIOES|BILOU|BMES|BMEOW|IO|IOB]
                                  [required]
  --output-labels [BIO|BIOES|BILOU|BMES|BMEOW|IO|IOB]
                                  [required]
  --help                          Show this message and exit.
```

## Repair

We can also apply repair methods to a file, creating an output file with only valid transitions. For example, we can run `seqscore repair --labels BIO --repair-method conlleval samples/invalid.bio samples/invalid_repair_conlleval.bio`, which will apply the conlleval repair method to the [samples/invalid.bio](https://github.com/bltlab/seqscore/blob/tutorial/samples/invalid.bio). The output would be the file [samples/invalid_repair_conlleval.bio](https://github.com/bltlab/seqscore/blob/tutorial/samples/invalid_repair_conlleval.bio):
```
This O
is O
a O
sentence O
. O

University B-ORG
of I-ORG
Pennsylvania I-ORG
is O
in O
West B-LOC
Philadelphia I-LOC
, O
Pennsylvania B-LOC
. O
```

If we want to apply the discard repair method, we can run `seqscore repair --labels BIO --repair-method discard samples/invalid.bio samples/invalid_repair_discard.bio` and the output is the file [samples/invalid_repair_discard.bio](https://github.com/bltlab/seqscore/blob/tutorial/samples/invalid_repair_discard.bio):
```
This O
is O
a O
sentence O
. O

University O
of O
Pennsylvania O
is O
in O
West B-LOC
Philadelphia I-LOC
, O
Pennsylvania B-LOC
. O
```

## Other commands

Other commands are still being documented, but here is a quick summary:
* `repair`: Apply a repair method to a file, creating an output file with
  only valid transitions.
* `convert`: Convert a file from one encoding to another.
* `count`: Output counts of chunks in the input file.
* `validate`: Check whether a file has any invalid transitions.


# FAQ

## Why can't I score output files that are in the format conlleval expects?

At this time, SeqScore intentionally does not support the "merged"
format used by conlleval where each line contains a token, correct
tag, and predicted tag:

```
University B-ORG B-ORG
of I-ORG I-ORG
Pennsylvania I-ORG I-ORG
is O O
in O O
West B-LOC B-LOC
Philadelphia I-LOC B-LOC
, O O
Pennsylvania B-LOC B-LOC
. O O
```

We do not support this format because we have found that creating
predictions in this format is a common source of errors in scoring
pipelines.


# Features coming soon!

* More documentation
* More error analysis tools


# Citation

If you use SeqScore, please cite
[SeqScore: Addressing Barriers to Reproducible Named Entity Recognition Evaluation](https://aclanthology.org/2021.eval4nlp-1.5/).

BibTeX:
```
@inproceedings{palen-michel-etal-2021-seqscore,
    title = "{S}eq{S}core: Addressing Barriers to Reproducible Named Entity Recognition Evaluation",
    author = "Palen-Michel, Chester  and
      Holley, Nolan  and
      Lignos, Constantine",
    booktitle = "Proceedings of the 2nd Workshop on Evaluation and Comparison of NLP Systems",
    month = nov,
    year = "2021",
    address = "Punta Cana, Dominican Republic",
    publisher = "Association for Computational Linguistics",
    url = "https://aclanthology.org/2021.eval4nlp-1.5",
    pages = "40--50",
}
```

# License

SeqScore is distributed under the MIT License.


# Development

For the latest development version, check out the `main` branch
(stable, but sometimes newer than the version on PyPI), or the `dev`
branch (latest, but less tested).

To install from a clone of this repository, use:
`pip install -e .`

## Setting up an environment for development

1. Create an environment: `conda create -y -n seqscore python=3.8`
2. Activate the environment: `conda activate seqscore`
3. Install dependencies: `pip install -r requirements.txt`
4. Install seqscore: `pip install -e .`
   x
