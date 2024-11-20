# ChineseWebText 2.0: Large-Scale High-quality Chinese Web Text with Multi-dimensional and fine-grained information
This directory contains the ChineseWebText2.0 dataset, and a new tool-chain called MDFG-tool for constructing large-scale and high-quality Chinese datasets with multi-dimensional and fine-grained information. Our ChineseWebText2.0 dataset is publicly available on huggingface (here).
## ChineseWebText2.0
- ### Dataset Overview
We have released the latest and largest Chinese dataset, ChineseWebText 2.0, which consists of 3.8 TB of data. Each text in the dataset is accompanied by a quality score, domain single-label and multi-label tags, as well as toxicity classification and scores, enabling LLM researchers to select data based on new quality thresholds.
- ### Data Example
   ```json
  {
   "text": "近日，黑龙江省高校校报协会第十四届学术年会暨校报工作交流研讨会在东北农业大学举行。我校10件新闻作品喜获2项一等奖，2项二等奖，6项三等奖……",
   "domain":
           {
             "single_label": "news",
             "multi_label": ["news", "education"]
           },
   "toxicity":
           {
             "label": 0,
             "score": 1.0347155694034882e-05
           },
   "quality_score": 0.96044921875
   }
    ```
   
- "text": 【string】Text content of data sample.
- "single_label": 【string】The highest probability label generated by the domain classification model.
- "multi_label": 【list】All labels generated by the domain classification model with probabilities higher than the threshold.
- "label": 【int】Toxicity label generated by toxicity classification models.
- "score": 【flaot】Toxicity score generated by toxicity classification model.
- "quality_score": 【float】Quality score generated by the quality evaluation model.

## MDFG-tool

### Introduction

We introduce a new toolchain, MDFG-tool (see Figure 1). We begin with the coarse-grained filtering module, which applies rule-based methods to clean the data, focusing on criteria such as text length and sensitive words to ensure data quality. After cleaning, we evaluate the text quality using a BERT-based model. This process generates a quality score, and by selecting an appropriate threshold, we can extract high-quality text data that meets our needs. Next, we use FastText for both single-label and multi-label classification of the cleaned data. Meanwhile, we conduct toxicity assessment. The FastText model is used to filter out toxic content and assign toxicity scores to each text. This scoring system allows researchers to set thresholds for identifying and selecting harmful texts for further training.

<div align="center">
  <img src=".\assets\structure.png" width="50%" />
</div>

### Environment Dependencies

```shell
transformers==4.31.0
scipy==1.11.1
numpy==1.24.3
jieba==0.42.1
zhconv==1.4.3
fasttext==0.9.2
```

### Stage 1: Preprocessing

This section focuses on extracting high-quality text from Chinese monolingual data by employing manually constructed rules to filter out violent, pornographic, and advertisement content, as well as erroneous characters. The detailed filtering rules are outlined as follows:

- #### Text Extraction

Extract text content from `jsonl` file after the data preparation stage.

- #### Data Length

To improve language model training, documents will be filtered out if they have an average line length of fewer than **10** characters or a total text length of less than 200 characters, as such short texts often lack meaningful context and semantic relevance.

- #### Proportion of Characters

We aim to create a high-quality simplified Chinese dataset from web data by eliminating traditional Chinese characters and removing texts with less than **30%** Chinese characters to ensure the dataset is suitable for training large language models.

- #### Sensitive Words

To prevent large language models from generating toxic content, a method is proposed where texts are analyzed for the occurrence of harmful words from a predefined list, and any text with more than **0.5** occurrences of such words per line is classified as toxic and removed from the training dataset.

- #### Internal duplication

To enhance training efficiency and model performance, a subsequent analysis using a 13-gram granularity is conducted to identify and filter out data samples where over **50%** of the character sequences are repetitive in each data entry.

Here is an example command to run the preprocessing stage:

```shell
python preprocess.py 
```

### Stage 2:  Quality Evaluation

In preprocessing procedure, we have used some handcrafted rules to remove the explicit noisy texts from our dataset. However, within the remaining data, there is still a considerable amount of low-quality text data, which cannot be filtered out with handcrafted rules. In order to extract the data of higher quality from them, in this section we further propose to design an evaluation models.

#### Stage 2.1:  BERTEval

#### 1. The Classification Results of Different Evaluation Models

<div align="center">
  <img src=".\assets\BERTEval.png" width="50%" />
</div>

#### 2. BERTEval Training and Inference

- Step 1: 2-stage Training

  ```shell
  python train.py # stage1  you can modify configs/base_config.json to set hyper-parameters
  python train_ust.py # stage2 you can modify configs/ust_config.json to set hyper-parameters
  ```

- Step 2: Split the previously processed CommonCrawl into multiple shards, where each shard is a JSON file. All shards for a single snapshot are stored in the same path. Refer to the example `util/text_separate.py`.

- Step 3: Run the Python inference script `pred.py` to split each text using delimiters such as newline `\n` or periods into complete paragraphs of a maximum length of 512. Predict the text quality score for each paragraph. The configuration can be modified using `config/pred_config.json`, with key parameters as follows:

  ```shell
  "data_path": ccnet data path
  "output_path": Path to store the scored data
  "num_workers": Number of CPU processes for data preprocessing
  "batch_size": BERT batch size
  "checkpoint": Model checkpoint path
  "tokenizer_path": Path to store BERT tokenizer
  "pretrained_model_path": Pre-trained BERT weights path
  ```

  Other parameters do not require modification. The processed text is stored in multiple JSONL files. Then, run

  ```shell
  python predict.py
  ```

  Step 4: Set the threshold value $T$ and retain text data with a quality threshold greater than $T$. Since the maximum input token limit for bert-base is 512, for longer texts, they are split into multiple text segments. For consecutive text segments in the same document with thresholds greater than $T$, the program automatically concatenates them. This functionality is implemented in the function `text_select_with_pred(file, score_threshold)` in `utils/util.py`.

  Usage:

  ```python
  file = "test\data\cleared0_0000.jsonl"
  score_threshold = 0.99
  selected_data = text_select_with_pred(file, score_threshold)

### Stage 3:  Domain Evaluation

#### 1. Composition of Domain Training and Test Data

