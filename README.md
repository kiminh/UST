# Uncertainty-aware Self-training

**UST** or **U**ncertainty-aware **S**elf-**T**raining is a method of task-specific training of pre-trainined language models (e.g., BERT, Electra, GPT) with only a few-labeled examples for the target classification task and large amounts of unlabeled data.

Our academic paper published as a spotlight presentation at NeurIPS 2020 describes the framework in details here: [Uncertainy-aware Self-training for Few-shot Text Classification](https://www.microsoft.com/en-us/research/publication/uncertainty-self-training-few-shot-bert/)

## Key Result

With only *20-30* labeled examples for each class for each task and large amounts of task-specific unlabeled data, UST performs within *3%* accuracy of fully supervised pre-trained language models fine-tuned on thousands of labeled examples with an aggregate accuracy of *91%* and improvement of upto *12%* over baselines (e.g., BERT) for text classification on benchmark datasets. It does not use any auxiliary resources like paraphrases or backtranslations.

## How it works

UST is a semi-supervised learning method that leverages pre-trained language models with stochastic regularization techniques and iterative self-training with student-teacher models. Specifically, it extends traditional self-training with three core components, namely: 

(i) *Masked model dropout for uncertainty estimation.* We adopt MC dropout (Gal and Ghahramani, 2016) as a technique to obtain uncertainty estimates from the pre-trained language model. In this, we apply stochastic dropouts after different hidden layers in the neural network model and approximate the model output as a random sample from the posterior distribution. This allows us to compute the model uncertainty in terms of the stochastic mean and variance of the samples with a few stochastic forward passes through the network. 

(ii) *Sample selection.* Given the above uncertainty estimates for a sample, we employ entropy-based measures to select samples that the teacher is most or least confused about to infuse for self-training corresponding to easy- and hard-entropy-aware example mining. 

(iii) *Confident learning.* In this, we train the student model to explicitly account for the teacher confidence by emphasizing on the low variance examples. All of the above components are jointly used for end-to-end learning.

## How to use the code

### Continued Pre-training on Task-specific Unlabeled Data (Optional)

For the few-shot learning setting with limited training labels, continued pre-training on task-specific unlabeled data *starting from available pre-trained checkpoints* is an effective mechanism to obtain a good base encoder to initialize the teacher model for UST. Code for continued pre-trainining with masked language modeling objective can be found in the original BERT repo here: [https://github.com/google-research/bert](https://github.com/google-research/bert). This requires invoking the `create_pretraining_data.py` and the `run_pretraining.py` scripts from the repo with additional instructions therein. This produces a new tensorflow checkpoint that can be used as the pre-trained checkpoint for UST.

You can use `transformers-cli` from [https://huggingface.co/transformers/converting_tensorflow_models.html](https://huggingface.co/transformers/converting_tensorflow_models.html) to convert tensorflow checkpoints (`ckpt`) to compatible checkpoints (`bin`) for HuggingFace transformers.

*Note that this continued pre-training step in optional for UST, but required to reproduce the results in the paper*. In absence of this step, UST uses the default pre-trained checkpoints for any pre-trained langauge model which also works very well in practise.

### HuggingFace Transformers as Base Encoders

UST is integrated with [HuggingFace Transformers](https://huggingface.co/transformers) which makes it possible to use any supported [pre-trained language model](https://huggingface.co/transformers/pretrained_models.html) as a base encoder.

### Training UST

UST requires 3 input files `train.tsv` and `test.tsv` with tab-separated (i) instances (e.g., SST and IMDB) or pairs of instances (e.g., MRPC and MNLI) and (ii) labels; and `transfer.txt` for the unlabeled instances of the corresponding task (all line-separated) in the data directory.

These are some standard set of arugments to run UST for the few-shot setting. Refer to `run_ust.py` for all the optional arugments and descriptions.

```
PYTHONHASHSEED=42 python run_ust.py 
--task $DATA_DIR 
--model_dir $OUTPUT_DIR 
--seq_len 128 
--sample_scheme easy_bald_class_conf 
--sup_labels 60 
--valid_split 0.5
--pt_teacher TFBertModel
--pt_teacher_checkpoint bert-base-uncased
--do_pairwise False
--N_base 5
--sup_batch_size 4
```

*Classification tasks:* Set `do_pairwise` to True for pairwise classification tasks like MRPC and MNLI. 

*Sampling schemes*: Supported `sample scheme`: `uniform`, `easy_bald_class_conf` (sampling easy examples with uncertainty given by Eqn. 7 in paper), `dif_bald_class_conf` (sampling difficult examples given by Eqn. 8). `conf` enables confident learning, whereas `class` enables class dependent exploration. Additionally, you can append `soft` to the above sampling scheme (e.g., `easy_bald_class_conf_soft`) for leveraging majority predictions from `T` stochastic forward passes that works well for settings involving many classes / labels.

*HuggingFace Transformers*: To use different pre-trained language models from HuggingFace, set `pt_teacher` and `pt_teacher_checkpoint` to corresponding model versions available from [https://huggingface.co/transformers/pretrained_models.html](https://huggingface.co/transformers/pretrained_models.html). A default set of pre-trained language models is available at ``huggingface_utils.py`.

*Training and validation*: `sup_labels` denote the total number of available labeled examples *for each class* for each task, where `valid_split` uses a fraction of those labels as validation set for early stopping. Set `sup_labels` to `-1` to use all training labels. Set `valid_split` to `-1` to use the available test data as the validation set.

*Initializing the teacher model*: To start with a good base encoder for the few-shot setting with very few labeled examples, UST uses different random seeds to initialize and fine-tune the teacher model `N_base` times and selects the best one to start the self-training process. This is not required when large number of labeled examples are available (correspondingly set `N_base` to `1`).

*Fine-tuning batch size*: Set `sup_batch_size` to a small number for few-shot fine-tuning (e.g., `4`) of the teacher model. In case of many training labels, set `sup_batch_size` to a higher value for faster training (e.g., `32`).

Self-training works for both low-data and high-data regime. For example, UST obtains *0.5* accuracy improvement for MNLI (mismatched) using all the available labeled examples (393K) to use for both training as well as the transfer set without using any additional unlabeled data. 

Standard set of arugments to run UST with all labeled examples (e.g., MNLI).

```
PYTHONHASHSEED=42 python run_ust.py 
--task $DATA_DIR/MNLI 
--model_dir $OUTPUT_DIR 
--seq_len 128 
--sample_scheme easy_bald_class_conf_soft 
--sup_labels -1 
--valid_split -1 
--sup_batch_size 32 
--do_pairwise 
--N_base 1
```
