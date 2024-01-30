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
### Taggers
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
### Deduplication
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
dolma -c dedupe_config.yaml tag
```
### Mixer
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
```
Run the file:
```bash
dolma -c mixer_config.yaml tag
```
