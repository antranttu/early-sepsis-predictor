# Early Sepsis Predictor
This is a project based on the [PhysioNet Computing in Cardiology Challenge 2019](https://physionet.org/content/challenge-2019/1.0.0/)

## Introduction
Sepsis is one of the leading causes of death in hospitals and the primary reason why patients are readmitted into medical care.
It is a dangerous medical condition that can kill the patient it affects, and it is due to the immune system’s disproportionate reaction to an infection.
In the U.S., nearly 1.7 million people develop sepsis and 270,000 people die from sepsis each year; over one third of people who die in U.S.
hospitals have sepsis ([CDC](https://www.cdc.gov/sepsis/datareports/index.html)). Internationally, an estimated 30 million people develop sepsis and 6 million people
die from sepsis each year; an estimated 4.2 million newborns and children are affected ([WHO](https://www.who.int/news-room/fact-sheets/detail/sepsis)).

Early detection and antibiotic treatment of sepsis are critical for improving sepsis outcomes, where each hour of delayed treatment has been associated
with roughly an 4-8% increase in mortality ([Kumar et al., 2006](https://www.ncbi.nlm.nih.gov/pubmed/16625125); [Seymour et al., 2017](https://www.ncbi.nlm.nih.gov/pubmed/28528569))

## Dataset
For this project, clinical data of ICU patients are provided by the challenge. The data for each patient are saved in a single pipe-delimited text file that
has a fixed header. Each row of a patient file represents a single hour's worth for all the measurements within that ICU-hour stay. These measurements include
vital signs, laboratory, and demographics values of 40 time-dependent variables. NaN indicates that the measurement is missing at a specific time interval.
In total, there are over 40,000 patient files. After concatenating entries from all the patients, there are about 1,5 million lines of hourly data in total.
For this project, only vital signs are used because they are accessible in most ICU facilities and are the medically recognized signs of sepsis.
Laboratory tests are costly, and they require time to confirm the diagnosis; therefore, they are not included as features for this project.

According to the Challenge, labels in the dataset already take the goal of predicting Sepsis 6 hours in advance into account. Therefore, the label for each hour
of patient data is 1 if the patient was diagnosed to develop sepsis 6 hours from the current hour. Summarized from the labels, we have a very imbalanced dataset
where only about 2% of the patterns result in sepsis. Original dataset is partitioned into training and test dataset respectively using stratified sampling to preserve
the class distribution in both of these datasets.

![](/images/imbalance.png)

## Objectives
By using clinical and patients’ data, this project aims to build a predictive model that will predict sepsis 6 hours before its onset in order for death-related
cases to be significantly reduced. As a result, the outcome of this project can help physicians diagnose sepsis in a timely manner so that effective treatment
can be prepared and administered in such a way that sepsis mortality rates could be mitigated. This project aims to propose a technique that includes additional
symptomatic and medical patterns that can help differentiate between sepsis and non-sepsis events. The class imbalance that exists between sepsis and non-sepsis
cases are also examined, an aspect that was not investigated by earlier research projects. The ultimate goal is to showcase that the instability of vital signs is
the main reason which results in the development of sepsis during a patient’s ICU stay.

## Data Cleaning

**Missing Value Imputation**

For this project, the last observation carried forward (LOCF) and the next observation carried backward (NOCB) mechanisms are used to fill in missing values.
LOCF and NOCB are two of the most popular methods in clinical trials, because they induce less bias to datasets compared to mean or median imputation.

LOCF is a common approach to handle missing data of temporal repeated measures data where some follow-up observations may be missing. In LOCF, a missing follow-up
value is imputed by that subject's previously observed value. In other words, if a patient has a missing value for one of their vital signs at a specific point
in time, the observed value of the previous hour in that vital sign is carried forward. On the other hand, NOCB works in the opposite direction by taking the first
available observation after the missing value and carrying it backward. LOCF and NOCB are applied to the dataset per patient; hence, values from one patient is
not filled to the subsequent patient in the dataset.

**Feature Engineering**

In this dataset, there is an absence of such variables that indicate how some vital signs would change within a specific time frame, which created the need
to utilize the temporal effect of this dataset. Therefore, this project utilized a sliding window of 6 hours to extract patterns of features that would differentiate
sepsis from non-sepsis events. It is important to note that after the feature engineering process, sepsis and non-sepsis events in this dataset are predicted
based on 6-hour patterns of data instead of original hourly data. Nine statistical measures are used to generate the new features that can capture the instability
of a specific vital sign over time, including mean, standard deviation, minimum, maximum, difference between the 6th and 5th hour (diff5), difference between the 5th
and 4th hour (diff4), difference between the 4th and 3rd hour (diff3), difference between the 3rd and 2nd hour (diff2), difference between the 2nd and 1st hour (diff1).
This 6-hour sliding window is applied on each patient from the beginning of their ICU stay until they are discharged from the ICU.

![](/images/sliding_window1.png)
![](/images/sliding_window2.png)
![](/images/sliding_window3.png)

**Dealing with imbalanced class distribution**

* Using a reliable performance metric: When having a severely imbalanced class such as the problem at hand, using accuracy as the metric for model comparison is
misleading. With 98% of total observations being non-sepsis patterns, if a model predicts every pattern to be non-sepsis, such model would achieve an accuracy
of 98%. Therefore, this project uses F1 score as the primary metric for performance comparison among different models. F1 score is a number that gives an
estimate of the balance that exists between precision and recall. Unlike other metrics, F1 score focuses on the performance of a classier on the
positive (minority class) only. It describes how good a model is at predicting the positive class.

* Threshold moving: The default probability threshold to classify an observation as the positive class is 0.5. In other words, if a predicted probability
for the observation by the model is greater than or equal to 0.5, this observation is classified as the positive class. Changing this decision threshold
is another approach to handle a severe class imbalance. This project aims to select a decision threshold that maximizes the F1 score.
In this case, all thresholds between 0.0 and 1.0 with a step size of 0.01 are tested, that is, 0.01, 0.02, 0.03, and so on to 0.99.
The optimal threshold that achieves the highest F1 score is selected for implementation.

* Data resampling: Most machine learning classifiers fail to cope with imbalanced classes as they are sensitive to the proportions of the different classes.
Consequently, these algorithms tend to favor the majority class, which may lead to misleading results. This is problematic when the minority class is the class
of interest. Resampling a dataset so that the proportion of classes are balanced to some extent is another method to tackle class imbalance. This project utilizes
under-sampling technique to conduct dataset resampling.

    * In some cases, seeking a balanced distribution for a severely imbalanced dataset can cause affected algorithms to overfit the minority class, leading to
    increased generalization error. As a result, the model can perform better on the training set, but worse on the test set as the test is not balanced.
    For this project, the ratio of majority class to minority class in the test dataset was kept at 98 to 2 (98:02) in order to reflect the real-world distribution.
    The training dataset, however, was resampled at different class distribution to evaluate which ratio would achieve the best performance on the test dataset.
    The different majority to minority class distribution experimented on the training dataset were 50:50, 80:20, 90:10, 95:05, 98:02, respectively.

## Models
Four algorithms are used to train different models, including Logistic Regression, Decision Tree, Random Forest, and Extreme Gradient Boosting (XGBoost). Each model is
trained with all feaures and with feature selection. Recursive Feature Elimination was used to select the most relevant features. The performances of the trained
models are then evaluated using the unseen test dataset, which was partitioned earlier. The model with the best performance on the test dataset is used as the model
for implementation for a web application. Each model is trained on different class distribution, and the **F1 score** of each model is reported as follows:

Class Distribution | Logistic Regression | Decision Tree | Random Forest | XGBoost
------------------ | ------------------- | ------------- | ------------- | -------
50:50 | 0.09 | 0.09 | 0.36 | 0.38
80:20 | 0.1 | 0.15 | 0.45 | 0.62
90:10 | 0.1 | 0.2 | 0.49 | 0.72
95:05 | 0.1 | 0.25 | 0.51 | 0.77
98:02 | 0.1 | 0.31 | 0.56 | **0.81**

## Code
* `data_processing.ipynb`: concatenating hourly records of all patients into 1 dataframe. Fill in missing values using LOCF and NOCB.
* `6hour_processing.ipynb`: using a 6-hour sliding window to extract patterns from the original vital signs per each patient, contains most of the data preparation and pre-processing.
* Models:
    * `log_reg.ipynb`: training of logistic regression
    * `dt.ipynb`: training of decision tree
    * `rf.ipynb`: training of random forest
    * `xgb.ipynb`: training of xgboost
    * `models_comparison.ipynb`: comparison of best model in each algorithm

## Results
All models were trained using a 10-fold cross-validation method that validated the results achieved by the models.
XGBoost with feature selection trained at 98:02 class distribution outperformed other models with an F1 score of 0.81, Kappa of 0.69,
AUROC of 0.94, and no false positives on the test dataset.

![](/images/f1_comparison.png)
![](/images/multiple_roc_curve.png)
