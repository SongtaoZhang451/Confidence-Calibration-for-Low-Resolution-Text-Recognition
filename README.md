# Confidence Calibration for Low-Resolution Text Recognition

This project evaluates whether scene text recognition models produce reliable confidence estimates when recognizing low-resolution images.

The study compares three pretrained text recognition models—**CRNN, ASTER, and MORAN**—on the TextZoom dataset. In addition to recognition accuracy, the project examines model calibration, overconfident errors, and the effects of visual degradation such as low contrast, blur, and complex backgrounds.

A lightweight post-processing method, **Platt scaling**, is then applied to determine whether confidence reliability can be improved without modifying the recognizers or changing their text predictions.

## Research Questions

This project investigates the following questions:

1. Do text recognizers make incorrect predictions with high confidence under low-resolution conditions?
2. Which visual conditions are most strongly associated with overconfident errors?
3. Can lightweight confidence calibration improve reliability without changing recognition accuracy?

## Dataset

The project uses the **TextZoom** dataset, which contains paired low-resolution and high-resolution scene-text images.

The main evaluation is conducted on the `test/medium` subset:

* Evaluation samples: **1,411**
* Calibration split: `train2`
* Prediction normalization: lowercase alphanumeric characters only
* Shared output alphabet: `a-z` and `0-9`

The repository includes `all_labels.csv`, which stores image paths, labels, and dataset split information. The TextZoom image files themselves are not included.

## Models

Three pretrained scene-text recognition models are evaluated:

### CRNN

CRNN combines:

* Convolutional neural networks for visual feature extraction
* Recurrent neural networks for sequence modeling
* Connectionist Temporal Classification for decoding

It serves as the baseline recognizer in this project.

### ASTER

ASTER uses:

* A spatial rectification network
* An attention-based sequence recognizer

The rectification stage is designed to improve recognition of distorted or irregular text.

### MORAN

MORAN combines:

* Multi-object rectification
* Attention-based sequence recognition

It provides another comparison for evaluating how architectural design affects accuracy and confidence reliability.

## Confidence Definition

Because the three recognizers use different decoders and vocabularies, their outputs are normalized into a shared format.

For each predicted word:

1. The maximum probability at each selected character position is extracted.
2. Unsupported characters are removed.
3. The geometric mean of the retained character probabilities is used as the word-level confidence score.

## Evaluation Metrics

| Metric                     | Description                                                                                    |
| -------------------------- | ---------------------------------------------------------------------------------------------- |
| Word Accuracy              | Proportion of predictions that exactly match the normalized ground-truth word                  |
| Expected Calibration Error | Difference between predicted confidence and empirical accuracy across confidence bins          |
| Brier Score                | Mean squared error between confidence and the binary correctness label                         |
| High-Confidence Error Rate | Fraction of samples that are incorrect while receiving confidence greater than or equal to 0.8 |

Lower ECE, Brier Score, and HCER values indicate better confidence reliability.

## Main Results

### Performance Before Calibration

| Model | Word Accuracy |        ECE | Brier Score |       HCER |
| ----- | ------------: | ---------: | ----------: | ---------: |
| ASTER |    **0.4111** | **0.2899** |  **0.2170** | **0.0914** |
| MORAN |        0.3664 |     0.3416 |      0.2334 |     0.0992 |
| CRNN  |        0.2055 |     0.4021 |      0.2739 |     0.1573 |

ASTER achieved the highest word accuracy and the strongest confidence reliability before calibration. CRNN had the lowest accuracy and the highest rate of high-confidence errors.

## Visual-Degradation Analysis

Three visual conditions are examined:

### Blur

Blur is measured using the variance of the Laplacian. Lower values indicate stronger blur.

The blur subgroup in the evaluation set contained very few samples, so the result is not sufficient to support a strong conclusion about the relationship between blur and overconfident errors.

### Low Contrast

Contrast is measured using the standard deviation of grayscale pixel intensity. Lower values indicate lower contrast.

All three recognizers showed higher HCER values in the low-contrast group, suggesting that low contrast is a consistent risk factor for overconfident recognition errors.

### Complex Background

Background complexity is estimated using a combination of:

* Canny edge density
* Normalized grayscale-image entropy

Complex backgrounds increased HCER for CRNN and MORAN. ASTER was comparatively more resistant and showed a slight decrease in HCER under this grouping.

## Confidence Calibration

Platt scaling is trained on the TextZoom `train2` split and applied to the `test/medium` predictions.

Calibration changes only the reported confidence values. It does not change the predicted text, so word accuracy remains unchanged.

### Results After Calibration

| Model | ECE Before |  ECE After | Brier Before | Brier After | HCER Before | HCER After |
| ----- | ---------: | ---------: | -----------: | ----------: | ----------: | ---------: |
| CRNN  |     0.4021 | **0.0797** |       0.2739 |  **0.0955** |      0.1573 | **0.0000** |
| ASTER |     0.2899 | **0.0682** |       0.2170 |  **0.1076** |      0.0914 | **0.0319** |
| MORAN |     0.3416 | **0.0692** |       0.2334 |  **0.0861** |      0.0992 | **0.0255** |

Platt scaling substantially reduced ECE, Brier Score, and HCER for all three recognizers.

An HCER value of zero does not mean that the recognizer made no errors. It means that none of its incorrect predictions remained above the selected confidence threshold after calibration.

## Repository Structure

```text
.
├── README.md
├── Extractor.ipynb
├── Project.ipynb
├── Report.pdf
└── all_labels.csv
```
* `Extractor.ipynb`: extracts paired LR/HR images and labels from the original TextZoom LMDB files
* `Project.ipynb`: model inference, confidence extraction, evaluation, visual-condition analysis, and calibration
* `Report.pdf`: full project report and discussion
* `all_labels.csv`: TextZoom labels, image paths, and dataset splits

## Pretrained Checkpoints

```text
.
├── aster_demo.pth
├── Moran_demo.pth
└── crnn.pth
```  
* `aster_demo.pth`: pretrained ASTER checkpoint used by the evaluation pipeline
* `Moran_demo.pth`: pretrained MORAN checkpoint used by the evaluation pipeline
* `crnn.pth`: pretrained CRNN checkpoint used by the evaluation pipeline

Download the Pretrained Checkpoints from:

* [Pretrained Checkpoints](https://drive.google.com/drive/folders/1LLJgzzbFV2312nnzjy_QIcQbAfn9onhb?usp=drive_link)

## Requirements

The main Python dependencies include:

```bash
pip install torch torchvision pandas numpy scipy scikit-learn \
    pillow opencv-python matplotlib jupyter
```

The original implementations and pretrained checkpoints for CRNN, ASTER, and MORAN must also be downloaded separately.

A CUDA-compatible GPU is recommended because the current notebook loads the recognizers on CUDA.

## Project Workflow

```text
TextZoom LMDB files
        |
        v
Extractor.ipynb
        |
        +--> Extract LR and HR PNG images
        |
        +--> Generate split-level labels.csv files
        |
        +--> Generate all_labels.csv
        |
        v
Project.ipynb
        |
        +--> Run CRNN, ASTER, and MORAN inference
        |
        +--> Normalize predictions and confidence scores
        |
        +--> Calculate accuracy and calibration metrics
        |
        +--> Analyze visual degradation groups
        |
        +--> Apply Platt scaling
        |
        v
Figures, tables, and final report
```

## External Repositories and Downloads

### TextZoom Dataset

Download the TextZoom dataset from:

* [WenjiaWang0312/TextZoom](https://github.com/WenjiaWang0312/TextZoom)

### Recognition Model Implementations

Download model repositories:

* [ASTER — ayumiymk/aster.pytorch](https://github.com/ayumiymk/aster.pytorch)
* [MORAN — Canjie-Luo/MORAN_v2](https://github.com/Canjie-Luo/MORAN_v2)
* [CRNN — meijieru/crnn.pytorch](https://github.com/meijieru/crnn.pytorch)

## Pretrained Checkpoints Included

| Model | Checkpoint File  | Approximate Size |
| ----- | ---------------- | ---------------: |
| ASTER | `aster_demo.pth` |            81 MB |
| MORAN | `Moran_demo.pth` |            78 MB |
| CRNN  | `crnn.pth`       |            32 MB |

Recommended local structure:

```text
checkpoints/
├── aster_demo.pth
├── Moran_demo.pth
└── crnn.pth
```

Update the paths in `Project.ipynb`:

```python
ASTER_CHECKPOINT = r"path/to/checkpoints/aster_demo.pth"
MORAN_CHECKPOINT = r"path/to/checkpoints/Moran_demo.pth"
CRNN_CHECKPOINT = r"path/to/checkpoints/crnn.pth"
```

## Running the Project

1. Download and extract the TextZoom dataset.
2. Download the original CRNN, ASTER, and MORAN implementations.
3. Obtain the corresponding pretrained model checkpoints.
4. Update the dataset, repository, and checkpoint paths near the beginning of the notebook.
5. Install the required Python packages.
6. Open the notebook:

```bash
jupyter notebook Project.ipynb
```

7. Run the notebook cells in order.

The current notebook contains local Windows paths. These paths must be replaced with paths that match the user's local environment.

## My Contributions

My work in this project includes:

* Building a unified evaluation pipeline for three different recognizers
* Normalizing predictions across different model vocabularies
* Defining a comparable word-level confidence score
* Calculating word accuracy, ECE, Brier Score, and HCER
* Identifying high-confidence recognition errors
* Measuring blur, contrast, and background complexity
* Comparing model reliability across visual-condition groups
* Training and evaluating Platt-scaling calibration models
* Producing reliability diagrams, comparison charts, and the final report

## Limitations

* The primary evaluation uses only the TextZoom `test/medium` subset.
* The blur subgroup contains very few observations, limiting the strength of the blur analysis.
* The visual-condition groups are based on handcrafted image statistics rather than manually verified degradation labels.
* Word confidence is constructed differently from the native confidence definitions of the original recognizers.
* The selected HCER threshold of 0.8 is useful for comparison but is still a design choice.
* Calibration performance should be validated on additional datasets and deployment conditions.

## Attribution

This repository does not claim authorship of the TextZoom dataset or the CRNN, ASTER, and MORAN model architectures and pretrained weights.

The original model implementations, papers, licenses, and copyright notices should be retained and followed. This repository focuses on the evaluation, comparison, visual-condition analysis, and confidence-calibration pipeline built around those models.

## References

1. Wang et al., *Scene Text Image Super-Resolution in the Wild*, ECCV, 2020.
2. Shi, Bai, and Yao, *An End-to-End Trainable Neural Network for Image-Based Sequence Recognition and Its Application to Scene Text Recognition*, IEEE TPAMI, 2017.
3. Shi et al., *ASTER: An Attentional Scene Text Recognizer with Flexible Rectification*, IEEE TPAMI, 2019.
4. Luo, Jin, and Sun, *MORAN: A Multi-Object Rectified Attention Network for Scene Text Recognition*, Pattern Recognition, 2019.

## Author

**Songtao Zhang**
