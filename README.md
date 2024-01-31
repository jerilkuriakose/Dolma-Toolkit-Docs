# Dolma Toolkit

## Installation
```bash
pip install dolma
```

## Dataset Source
```
provisioningte0256624006 / llm-data / dolma / data
```

## Example
For this example I will be using `wiki-en-simple` dataset.

### Download data
```bash
azcopy cp <source> <destination> --recursive=true
```
Example
```bash
azcopy cp "https://provisioningte0256624006.blob.core.windows.net/llm-data/dolma/data/wiki-en-simple?<token>" /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets --recursive=true
```
The data will be now found in the following directory:
```bash
/home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets
```
The following will be the directory structure
```plain-text
|-- wiki-en-simple/
    |-- en_simple_wiki-0000.json        (10GB)
    |-- en_simple_wiki-0001.json        (5GB)
```
Whereas to run the `dolma toolkit`, the following should be the directory structure
```plain-text
|-- dataset-name/
    |-- documents/
        |-- 2019-09/
            |-- 0933_uk_all.jsonl.gz        (1GB)
            |-- 0933_vi_all.jsonl.gz        (1GB)
            |-- 0106_uk_all.jsonl.gz        (1GB)
            |-- 0106_vi_all.jsonl.gz        (1GB)
        |-- 2019-08/
            |-- ...
    |-- attributes/
        |-- toxicity-0/
            |-- 2019-09/
                |-- 0933_uk_all.jsonl.gz    (..MB)
                |-- 0933_vi_all.jsonl.gz    (..MB)
                |-- 0106_uk_all.jsonl.gz    (..MB)
                |-- 0106_vi_all.jsonl.gz    (..MB)
            |-- 2019-08/
                |-- ...
        |-- paragraph_duplicates/
            |-- ...
```
So in our case it should be the following:
```plain-text
|-- wiki-en-simple/
    |-- documents/
        |-- en_simple_wiki-0000.json        (10GB)
        |-- en_simple_wiki-0001.json        (5GB)
```
Run the following command to convert it into the dolma format:
```bash
cd /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple
mkdir documents
mv -t documents/ en_simple_wiki-0000.json en_simple_wiki-0001.json
```

### 1. Taggers
Run the built-in taggers on the downloaded documents. We can used a `YAML` file to configure the required parametes.

#### Configuration
The following is the `YAML` file required to run the taggers: `tag_config.yaml`
```yaml
# the place where the dataset is downloaded
documents:
  - /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/*

# name of the experiment
# this will create a folder named "exp" inside the attributes folder
# and the tagged documents are found inside that
experiment: exp

# run the list of taggers on all documents in the specified S3 paths
taggers:
  - random_number_v1
  - cld2_en_paragraph_with_doc_score_v2
  - ft_lang_id_en_paragraph_with_doc_score_v2
  - char_length_with_paragraphs_v1
  - whitespace_tokenizer_with_paragraphs_v1

# to make it run in parallel
processes: 8
```
Run the file:
```bash
dolma -c tag_config.yaml tag
```
Note: If the following error is encountered :
```bach
ValueError: Invalid mode: 'rt'
```
Kindly modify the source code. Change `rt` to `r`

*Output*:
```bash
(dolma_env) azureuser@jnkuriakose2:~/cloudfiles/code/Users/JNKuriakose/dolma_stuffs$ dolma -c tag_config.yaml tag
[nltk_data] Downloading package punkt to /home/azureuser/nltk_data...
[nltk_data]   Unzipping tokenizers/punkt.zip.
debug: false
destination: null
documents:
- /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/*
dryrun: false
experiment: exp
ignore_existing: false
processes: 4
profile:
  enable: false
  lines: 100
  output: null
  sort_key: tottime
  steps: null
tagger_modules: []
taggers:
- random_number_v1
- cld2_en_paragraph_with_doc_score_v2
- ft_lang_id_en_paragraph_with_doc_score_v2
- char_length_with_paragraphs_v1
- whitespace_tokenizer_with_paragraphs_v1
work_dir:
  input: null
  output: null
Found 2 files to process
Downloading …/supervised-models/lid.176.bin ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:02 131.3/131.3 MB
documents: 6.11Md [2:21:55, 718d/s]
files: 2.00f [2:21:55, 4.26ks/f]/s]
```

### 2. Conversion Step (only for uncompressed dataset)
The dataset available in Azure has been extracted. Whereas for deduplication Dolma toolkit requires the files to be in `.gz` format. So the dataset needs to be compressed again.
```bash
cd /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/
pigz */*.json
```
Need to do the same for the attributes created or else will result in error during the `mix` stage:
```bash
cd /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/attributes/exp
pigz */*.json
```

### 3. Deduplication
To deduplicate a set of documents at the attribute or paragraph level using a Bloom filter.

Just like taggers, `dolma dedupe` will create a set of attribute files, corresponding to the specified input document files. The attributes will identify whether the entire document is a duplicate (based on some key), or identify spans in the text that contain duplicate paragraphs.

#### Configuration
Deduplicate using Paragraph: `dedupe_config.yaml`
```yaml
# Paths to the documents to be deduplicated
documents:
  - /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/*

# dedupe details
dedupe:
  name: dedupe_paragraphs
  paragraphs:
    # Name of the output field in the tagger
    attribute_name: bff_duplicate_paragraph_spans
  # empty documents/paragraphs will be skipped
  skip_empty: true

# bloom filter details
bloom_filter:
  # Path where to read/write the bloom filter file to/from
  file: /tmp/deduper_bloom_filter.bin
  # Estimated number of documents to be added to the bloom filter
  estimated_doc_count: 6_000_000
  # Desired false positive rate
  desired_false_positive_rate: 0.0001
  # If true, the bloom filter will be read from the file and not updated
  read_only: false

# to make it run in parallel
processes: 8
```
*Output*
```bash
(dolma_env) azureuser@jnkuriakose2:~/cloudfiles/code/Users/JNKuriakose/dolma_stuffs$ dolma -c dedupe_config.yaml dedupe
bloom_filter:
  desired_false_positive_rate: 0.0001
  estimated_doc_count: 6000000
  file: /tmp/deduper_bloom_filter.bin
  read_only: false
  size_in_bytes: 0
dedupe:
  name: dedupe_paragraphs
  paragraphs:
    attribute_name: bff_duplicate_paragraph_spans
  skip_empty: true
documents:
- /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/*
processes: 8
work_dir:
  input: /tmp/dolma-input-vdcmgmll
  output: /tmp/dolma-output-q_enf2zn
[2024-01-31T07:41:58Z INFO  dolma::bloom_filter] Loading bloom filter from "/tmp/deduper_bloom_filter.bin"...
[2024-01-31T07:41:58Z INFO  dolma::deduper] Writing attributes for /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/en_simple_wiki-0001.json.gz to /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/attributes/dedupe_paragraphs/en_simple_wiki-0001.json.gz.tmp
[2024-01-31T07:41:58Z INFO  dolma::deduper] Writing attributes for /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/en_simple_wiki-0001.json.gz to /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/attributes/dedupe_paragraphs/en_simple_wiki-0001.json.gz.tmp
[2024-01-31T07:41:58Z INFO  dolma::deduper] Writing attributes for /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/en_simple_wiki-0000.json.gz to /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/attributes/dedupe_paragraphs/en_simple_wiki-0000.json.gz.tmp
[2024-01-31T07:41:58Z INFO  dolma::deduper] Writing attributes for /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/en_simple_wiki-0000.json.gz to /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/attributes/dedupe_paragraphs/en_simple_wiki-0000.json.gz.tmp
[2024-01-31T07:43:32Z INFO  dolma::deduper] Keeping local file "/home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/en_simple_wiki-0001.json.gz" after deduping...
[2024-01-31T07:44:58Z INFO  dolma::deduper] Keeping local file "/home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/en_simple_wiki-0000.json.gz" after deduping...
[2024-01-31T07:44:58Z INFO  dolma::deduper] Writing bloom filter to "/tmp/deduper_bloom_filter.bin"...
[2024-01-31T07:44:58Z INFO  dolma::deduper] Bloom filter written.
[2024-01-31T07:44:58Z INFO  dolma::deduper] Done!
```
Deduplicate using URL: `dedupe_config.yaml`
```yaml
# Paths to the documents to be deduplicated
documents:
  - /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/*

# dedupe details
dedupe:
  name: dedupe_by_url
  documents: 
    attribute_name: bff_duplicate_url
    key: "$.metadata.url"

# bloom filter details
bloom_filter:
  # Path where to read/write the bloom filter file to/from
  file: /tmp/deduper_bloom_filter.bin
  # Estimated number of documents to be added to the bloom filter
  estimated_doc_count: 6_000_000
  # Desired false positive rate
  desired_false_positive_rate: 0.0001
  # If true, the bloom filter will be read from the file and not updated
  read_only: false

# to make it run in parallel
processes: 8
```
Run the file:
```bash
dolma -c dedupe_config.yaml dedupe
```

### 4. Mixer
Combines data from multiple sources into a unified output. Merges the named attributes and applies the configured filters.
#### Configuration
Mixer example: `mixer_config.yaml`
```yaml
# mix command operates on one or more stream; each can correspond to a different data source
# and can have its own set of filters and transformations
streams:
  # name of the stream; this will be used as a prefix for the output files
  - name: getting-started
    
    # the documents to mix; glob pattern is used to match all documents
    documents:
      - /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/*.gz
    
    # this is the directory where the output will be written
    # note how the toolkit will try to create files of size ~1GB
    output:
      path: /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/mixer/documents
      max_size_in_bytes: 1_000_000_000
    
    # the name of the tagger experiment and dedupe attribute
    attributes:
      # load the attributes from the taggers
      - exp
      # load the attributes from the deduper
      - bff_duplicate_paragraph_spans
    
    # filers remove or include whole documents based on the value of their attributes
    filter:
      include:
        # Include all documents with length less than 100,000 whitespace-separated words
        - "$.attributes[?(@.exp__whitespace_tokenizer_with_paragraphs_v1__document[0][2] < 100000)]"
      exclude:
        # Remove any document that is shorter than 50 words
        - "$.attributes[?(@.exp__whitespace_tokenizer_with_paragraphs_v1__document[0][2] < 50)]"
        # Remove any document whose total English fasttext score is below 0.5
        - "$.attributes[?(@.exp__ft_lang_id_en_paragraph_with_doc_score_v2__doc_en[0][2] <= 0.5)]"
        # Remove all documents that contain a duplicate paragraph
        - "$@.attributes[?(@.bff_duplicate_paragraph_spans && @.bff_duplicate_paragraph_spans[0] && @.bff_duplicate_paragraph_spans[0][2] >= 1.0)]"

    # span replacement allows to replace spans of text with a different string
    span_replacement:
      # remove paragraphs whose not-English cld2 socre is below 0.9 in a document
      - span: "$.attributes.exp__cld2_en_paragraph_with_doc_score_v2__not_en"
        min_score: 0.1
        replacement: ""

# to make it run in parallel
processes: 8
```
Run the file:
```bash
dolma -c mixer_config.yaml tag
```
*Output*
```bash
(dolma_env) azureuser@jnkuriakose2:~/cloudfiles/code/Users/JNKuriakose/dolma_stuffs$ dolma -c mixer_config.yaml mix
processes: 8
streams:
- attributes:
  - exp
  - dedupe_paragraphs
  documents:
  - /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/*.gz
  filter:
    exclude:
    - $.attributes[?(@.exp__whitespace_tokenizer_with_paragraphs_v1__document[0][2]
      < 50)]
    - $.attributes[?(@.exp__ft_lang_id_en_paragraph_with_doc_score_v2__doc_en[0][2]
      <= 0.5)]
    - $@.attributes[?(@.bff_duplicate_paragraph_spans && @.bff_duplicate_paragraph_spans[0]
      && @.bff_duplicate_paragraph_spans[0][2] >= 1.0)]
    include:
    - $.attributes[?(@.exp__whitespace_tokenizer_with_paragraphs_v1__document[0][2]
      < 100000)]
  name: getting-started
  output:
    max_size_in_bytes: 1000000000
    path: /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/mixer/documents
  span_replacement:
  - min_score: 0.1
    replacement: ''
    span: $.attributes.exp__cld2_en_paragraph_with_doc_score_v2__not_en
work_dir:
  input: /tmp/dolma-input-z5tl71sa
  output: /tmp/dolma-output-myqpicl2
[2024-01-31T09:02:55Z INFO  dolma::shard] Computing shards for stream getting-started...
[2024-01-31T09:02:55Z INFO  dolma::shard] Splitting 2 files for getting-started into 2 shards
[2024-01-31T09:02:56Z INFO  dolma::mixer] Building output "/home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/mixer/documents/getting-started-0000.json.gz"...
[2024-01-31T09:02:56Z INFO  dolma::mixer] Building output "/home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/mixer/documents/getting-started-0001.json.gz"...
[2024-01-31T09:02:56Z INFO  dolma::shard] Merging /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/en_simple_wiki-0000.json.gz into /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/mixer/documents/getting-started-0000.json.gz
[2024-01-31T09:02:56Z INFO  dolma::shard] Merging /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/en_simple_wiki-0001.json.gz into /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/mixer/documents/getting-started-0001.json.gz
[2024-01-31T09:11:38Z INFO  dolma::shard] Dropped 2928291 of 2928291 documents from /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/en_simple_wiki-0001.json.gz
[2024-01-31T09:14:48Z INFO  dolma::shard] Dropped 3159681 of 3181719 documents from /home/azureuser/cloudfiles/code/Users/JNKuriakose/dolma_stuffs/datasets/wiki-en-simple/documents/en_simple_wiki-0000.json.gz
[2024-01-31T09:14:48Z INFO  dolma::mixer] Done!
```
