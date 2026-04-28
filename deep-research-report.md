# Executive Summary  
Cybersecurity flow datasets such as CIC-IDS2017 and CSE-CIC-IDS2018 provide labeled network flows for training intrusion-detection models. These datasets are public (via the Canadian Institute for Cybersecurity, UNB) and include both benign traffic and a variety of attack scenarios. CIC-IDS2017 contains ~2.83 million flows over 5 days, with 79 fields (78 numeric + Label)【40†L13-L17】【50†L155-L163】. It covers attacks like DoS (Hulk, GoldenEye, slowloris, slowhttptest), DDoS (HOIC/LOIC), port scans, brute-forces, web exploits (XSS, SQLi, brute-force), infiltration, Heartbleed, and botnet【50†L155-L163】【50†L171-L176】. CSE-CIC-IDS2018 (AWS public dataset) organizes flows by day and machine, with ~80+ features (including FlowID, IPs, ports, protocol, plus 80 traffic metrics)【58†L613-L620】【72†L55-L63】. It includes seven broad attack scenarios (Brute-force, Heartbleed, Botnet, DoS, DDoS, Web, Infiltration)【35†L1-L4】, each with subtypes (e.g. multiple DDoS variants, web-XSS/SQLi). Both datasets are highly imbalanced (benign flows dominate, ~80% in CIC-IDS2017【50†L155-L163】). Preparing these for fine-tuning a CySecBERT model requires careful preprocessing: mapping labels (binary vs multi-class), cleaning features, scaling, handling categorical fields, and balancing classes. This report details official sources, dataset schemas, label taxonomies, common issues (e.g. header duplication【55†L293-L299】, CICFlowMeter bugs【28†L99-L108】【28†L121-L129】), and provides end-to-end pipelines (data splitting, serialization) with code pseudocode examples. We also recommend evaluation metrics and hyperparameters for CySecBERT fine-tuning.

## Dataset Sources and Versions  
The CIC-IDS series are maintained by UNB’s Canadian Institute for Cybersecurity (CIC)【71†L398-L402】. The **CIC-IDS2017** dataset is downloadable (registration required) via the CIC website【71†L398-L402】. It was introduced by Sharafaldin *et al.* (ICISSP 2018) and has a companion “MachineLearningCSV” collection of flows【40†L13-L17】. An alternate “BCCC-CIC-IDS2017” version is hosted by York University【71†L398-L402】. The **CSE-CIC-IDS2018** (often called “IDS-2018”) is a joint CSE–CIC project. Its flows and logs (split by day and host) are publicly available on AWS Open Data (bucket `s3://cse-cic-ids2018/`); UNB provides AWS CLI sync instructions【58†L639-L644】. Kaggle also hosts mirror CSV versions (e.g. “IDS Intrusion CSVs (CSE-CIC-IDS2018)”). For completeness, other flow datasets include **UNSW-NB15** (UNSW Canberra, 2015) with 2.54M records and 49 features【24†L35-L43】, plus older benchmarks like NSL-KDD; these can serve as alternatives but are not focus here. In summary:

| Dataset               | Source                            | Year | Size (flows)            | #Features | Attack Classes                                |
|-----------------------|-----------------------------------|------|-------------------------|-----------|-----------------------------------------------|
| **CIC-IDS2017**       | CIC@UNB (MachineLearningCSV)      | 2017 | 2,830,743【50†L155-L163】 | 79 (78 num + Label)【40†L13-L17】 | Benign + {DoS Hulk, DoS GoldenEye, DoS slowloris, DoS slowhttptest, DDoS-HOIC, DDoS-LOIC, PortScan, FTP-Patator (Brute FTP), SSH-Patator (Brute SSH), Web-BruteForce, Web-XSS, Web-SQLi, Infiltration, Heartbleed, Botnet}【50†L155-L163】【50†L171-L176】 |
| **CSE-CIC-IDS2018**   | CIC@UNB (AWS Public Dataset)      | 2018 | multi-million (50 machines, 5 days) | ~80+【58†L613-L620】 | Benign + {Brute-force, Heartbleed, Botnet, DoS, DDoS, Web, Infiltration}【35†L1-L4】 (detailed variants shown in [55]) |
| **UNSW-NB15**         | UNSW Canberra                     | 2015 | 2,540,044【24†L35-L43】 | 49【24†L35-L43】 | 9 attack types (Fuzzers, Analysis, Backdoors, DoS, Exploits, Generic, Reconnaissance, Shellcode, Worm)【24†L35-L43】 + Benign |

## Dataset Schemas and Labels  
Both CIC datasets are flow-based CSVs. Each record typically includes identifiers (FlowID, Source/Dest IP, Source/Dest Port, Protocol) and ~80 traffic metrics (durations, packet counts, byte counts, flags, etc.)【72†L55-L63】【58†L613-L620】. For example, CICFlowMeter features in IDS2018 include “Flow duration, total packets forward/backward, byte counts, packet size statistics, inter-arrival times, TCP flag counts, window bytes, subflow stats, flow active/idle times”【16†L517-L526】【40†L13-L17】. The **Label** column is categorical. In CIC-IDS2017 it contains values like “BENIGN,” “DoS Hulk,” “PortScan,” “FTP-Patator,” “Web Attack – XSS,” etc.【50†L155-L163】【50†L171-L176】. In CSE-CIC-IDS2018, labels include “Benign” and the seven scenario names (e.g. “FTP-BruteForce,” “DDOS attack-HOIC,” “DoS attacks-GoldenEye,” “Web attacks – SQL Injection,” etc.)【55†L277-L285】【35†L1-L4】. UNSW-NB15 has fields like “attack_cat” with 9 categories (listed above). All feature columns are numeric (float or integer). The few string/categorical fields (e.g. Protocol) may need encoding.

### Class Distributions and Issues  
All these datasets are heavily imbalanced. In CIC-IDS2017, ~2.27M of 2.83M flows (~80%) are benign【50†L155-L163】; large classes include “DoS Hulk” (~231K) and “PortScan” (~158K), while many classes (e.g. Heartbleed, SQLi) have only a few dozen samples【50†L155-L163】【50†L171-L176】. CSE-CIC-IDS2018 similarly has majority benign traffic, with each attack class typically <10% of data. UNSW-NB15 reports ~86% benign (estimating from 175K train, 82K test vs total 2.54M). These imbalances motivate careful sampling (see below).  

Common data issues include missing or malformed values. In practice, CIC flows from pcaps have no NaNs (missing counters are usually zero) unless extraction failed. However, user-hosted CSVs (e.g. Kaggle copies) can have quirks. For example, one Kaggle release of CSE-CIC-IDS2018 had duplicate header rows in some files, causing pandas to read all columns as strings【55†L293-L299】. This must be fixed by removing extra header lines before parsing. Also, researchers noted that CICFlowMeter v3 had bugs: in original CIC-IDS2017 flows, TCP flag count fields were mostly 0/1 due to an extractor bug, and “Fwd Header Length” was duplicated【28†L99-L108】【28†L121-L129】. These bugs have been fixed in updated tool versions, but one should verify and, if needed, drop duplicate columns or recalc flags. 

## Preprocessing Pipeline  

- **Label Mapping:** For binary classification, map `Label` to {0=Benign, 1=Malicious}, grouping all attack types together. For multi-class, retain each attack category (e.g. “DoS Hulk,” “Web-XSS”) or merge similar ones. Common strategies merge subtypes: e.g. combine “Web-XSS” and “Web-SQLi” into “WebAttack,” or “DDoS-HOIC/LOIC” into one “DDoS” class. The precise taxonomy depends on the task. In all cases, encode labels as integers or one-hot for model training.

- **Feature Selection:** Use the flow features generated by CICFlowMeter. One should drop identifier columns (FlowID, IPs, ports) since they do not generalize. (If using any categorical like Protocol, one-hot encode or include as text.) Domain papers often use all 78 numeric features【16†L517-L526】. Optional selection/PCA can be applied (some studies do PCA)【40†L29-L32】, but modern BERT models can handle many features.  

- **Categorical vs. Numeric:** Most CIC features are numeric. For any categorical fields (e.g. protocol names), convert to numeric via one-hot or embedding. Since CySecBERT is a language model, another approach is to “textualize” flows: e.g. concatenate features into a string with labels (“protocol TCP, src_port 80, flags 18, ...”). However, one can also tokenize numeric values directly as text tokens. Ensure type consistency. 

- **Normalization/Scaling:** Numeric features span wide ranges (e.g. durations, byte counts). Apply scaling (e.g. StandardScaler or MinMax) to improve training. Log-transform features with heavy tails (like packet counts) to reduce skew. In token-based BERT input, scaling may matter less if treating numbers as distinct tokens, but normalization aids model convergence if embedding numeric tokens.  

- **Missing Values:** Typically, CIC flow files have no missing entries. If any NaNs appear (e.g. due to parsing), impute them (zero or column mean) or drop affected flows. Verify that categorical encoding did not produce blanks.  

- **Outlier Handling:** Very large flows (e.g. due to the 120-second timeout splitting) can be outliers. You may clip extreme values or remove top percentile flows when appropriate. Alternatively, use robust scaling.

- **Flow Aggregation/Sessionization:** The raw dataset is already flow-level. Optionally, one can aggregate flows into sessions by 5-tuple or time-window features (e.g. count of flows per host per time bin). For CySecBERT fine-tuning, we treat each flow as an “example.” If richer context is needed, grouping flows by session (e.g. all flows of a TCP connection) could be done prior to tokenization.

- **Sampling/Oversampling:** Due to imbalance, training on raw data may bias toward benign. Strategies include under-sampling the majority, or over-sampling minorities. SMOTE (Synthetic Minority Over-sampling TEchnique) has been used successfully【40†L29-L32】. For example, after splitting train/test, apply SMOTE on the training set to generate synthetic attack samples. Alternatively, use class-weighting or create balanced batches. Oversampling should be done only on the training data to avoid test leakage.

The flowchart below outlines a typical preprocessing pipeline:

```mermaid
flowchart LR
    A[Raw PCAPs / CSVs] --> B{Parsing and Merge}
    B --> C[CICFlowMeter Extraction] 
    B --> D[Download CSV (CIC)]
    C & D --> E[Combined Flow CSV]
    E --> F[Data Cleaning (remove duplicates, extra headers)【55†L293-L299】]
    F --> G[Drop IDs (FlowID, IPs); encode categorical (Protocol)]
    G --> H[Impute/Missing (fill 0)]
    H --> I[Feature Scaling (Standardize/Log)]
    I --> J[Label Mapping (binary or multi)]
    J --> K[Train/Val/Test Split (time-based or stratified)【50†L155-L163】【50†L177-L185】]
    K --> L[CySecBERT Tokenization]
    L --> M[Model Fine-tuning]
    M --> N[Evaluation (Accuracy, F1, AUC)]
```

## Data Splits and Serialization  
When splitting data, temporal splits are often recommended to simulate realistic deployment. For CIC-IDS2017, for example, researchers split by days or held out certain attack classes entirely for an “unknown attack” test set【50†L177-L185】. A common scheme is training on days 1–n-1 and testing on day n. Otherwise, one can use random stratified splits, ensuring each class is represented (stratified K-fold). Always reserve a held-out test set (never use test data in preprocessing). For cross-validation, use time-series aware folds or repeated random splits on different days.

Data should be serialized for model input. HuggingFace Datasets accepts CSV, JSONL, or TFRecord. For example, save each example as a line of JSON with fields {"text": "...", "label": ...}. For CSV/JSON, include columns like `features` and `label`. TFRecord is useful for TensorFlow pipelines. A pseudocode example to create a HF dataset:

```python
from datasets import Dataset
import pandas as pd
# After preprocessing into pandas:
df = pd.read_csv("flows_prepped.csv")   # with columns ['text','label']
dataset = Dataset.from_pandas(df)       # HuggingFace Dataset
tokenizer = AutoTokenizer.from_pretrained("CySecBERT")
def tokenize_example(ex):
    enc = tokenizer(ex["text"], padding="max_length", truncation=True)
    enc["label"] = ex["label"]
    return enc
tokenized_ds = dataset.map(tokenize_example, batched=True)
```

Each `text` can be a concatenation of selected features (e.g. `f"dur {flow_duration} pkt_s {packets/sec} flags {flag_str} ..."`). Alternatively, one might input raw numeric tokens (e.g. `"120.0 3.0 18.0 ..."`). Ensure consistent tokenization. After tokenization, create PyTorch/TF `Dataset` objects as usual. CySecBERT expects token IDs and attention masks, plus labels.

## Evaluation Metrics and Validation  
**Binary classification:** Use accuracy, precision, recall, and F1-score, but also ROC-AUC (or PR-AUC) to account for imbalance. Specifically, report **F1** (especially for the malicious class) and **False Positive Rate** (important in IDS). Confusion matrices and precision-recall curves are informative. **Multi-class:** report overall accuracy and per-class precision/recall/F1. Macro-averaged F1 (averaging classes) is useful when classes are imbalanced. Also track per-class recall to ensure small attack classes are detected. 

For validation, use stratified k-fold or repeated random splits for hyperparameter tuning, but final evaluation should use a time-based or “unseen attack” split. In one study, certain attacks were held out of training to form an “unknown attack” test set【50†L177-L185】. This assesses generalization to new threats. Always avoid temporal leakage: e.g., do not train on flows that occurred after test flows.

## Tables: Dataset Comparisons  

| **Dataset**        | **Year** | **Flows**           | **#Features** | **Attack Classes (besides Benign)**                                   | **Benign %** |
|--------------------|----------|---------------------|---------------|-----------------------------------------------------------------------|--------------|
| CIC-IDS2017【40†L13-L17】【50†L155-L163】 | 2017     | 2,830,743          | 79 (78 numeric + Label)【40†L13-L17】 | DoS Hulk, GoldenEye, slowloris, slowhttptest; DDoS-HOIC, LOIC; PortScan; FTP-Patator; SSH-Patator; Web-BruteForce; Web-XSS; Web-SQLi; Infiltration; Heartbleed; Botnet【50†L155-L163】【50†L171-L176】 | ~80%【50†L155-L163】 |
| CSE-CIC-IDS2018【35†L1-L4】【58†L613-L620】 | 2018     | multi-million (50 hosts) | ~80+            | Botnet; FTP/SSH-BruteForce; Web (XSS, SQLi, etc.); DoS (Hulk, Slowloris, SlowHTTP, GoldenEye); DDoS (HOIC, LOIC-UDP/HTTP); Infiltration; Heartbleed【35†L1-L4】【55†L277-L284】 | Majority (high) |
| UNSW-NB15【24†L35-L43】     | 2015     | 2,540,044          | 49【24†L35-L43】  | 9 types: Fuzzers, Analysis, Backdoors, DoS, Exploits, Generic, Reconnaissance, Shellcode, Worm【24†L35-L43】                    | ~86%        |

## Hyperparameters and Augmentation  

For CySecBERT fine-tuning, typical BERT hyperparameters apply. Recommended batch sizes are ~16–32 (adjustable to GPU memory). Learning rate ~2e-5 to 5e-5 is a good starting point, with a linear warmup; many use 3 epochs, though larger datasets can go to 5–10 epochs. (The CySecBERT pretraining itself used ~2e-5 for 30 epochs【68†L7-L10】; fine-tuning may converge faster.) Use AdamW optimizer, possibly with weight decay (e.g. 0.01). Because of class imbalance, you may incorporate class weighting or a lower learning rate for minority classes. Validate these settings via a hold-out or cross-validation split.

Data augmentation in tabular flow data is challenging but possible. Besides SMOTE/ADASYN oversampling (which effectively *generate* new samples)【40†L29-L32】, one can add small Gaussian noise to numeric features or use mixup/interpolation between flows of the same class. More advanced: use a GAN or VAE trained on minority classes to synthesize realistic flows (see recent work on NID augmentation). Random feature dropout or slight scaling of payload statistics can also increase robustness. Always ensure augmented data stays physically plausible (e.g. ports remain valid integers). 

**Note:** All details not explicitly specified above (e.g. CySecBERT’s expected input format) should be decided based on downstream requirements. For example, if CySecBERT was pretrained on textual cyber data, then mapping flows to text tokens is needed. If it can accept arbitrary tokens, one could directly map numeric features to string tokens. Always document these choices.

**References:** We used official dataset documentation and recent studies to compile this pipeline. Key sources include the UNB dataset pages【40†L13-L17】【58†L613-L620】, extended analyses of CIC-IDS2017【28†L99-L108】【28†L121-L129】, and sample code notebooks【55†L293-L299】. For example, Sharafaldin *et al.* (2018) is cited for CIC-IDS2017, while UNSW-NB15 is documented by Moustafa *et al.*【24†L35-L43】. We also reference recent usage of these datasets (e.g. class counts【50†L155-L163】【50†L171-L176】) to illustrate distributions.  

