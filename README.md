# sparkHospital

## Overview

This project focused on determining patient outcomes based on data taken during their first twenty-four hours in an Intensive Care Unit (ICU). We aimed to develop a model that will accurately predict a patient’s survival based on this information. Intensive Care Units are meant for individuals who are extremely sick or hurt and require intensive treatment and close monitoring. Patients in the ICU are typically those who would not be able to survive without the team of staff and the resources available such as ventilators, monitoring equipment, IVs, feeding tubes, and drains and catheters (https://www.nhs.uk/conditions/intensive-care/). Therefore, being able to understand the likelihood of a person’s survival is important as that can help staff allocate resources more efficiently and effectively. If a person is likely to survive, it may be possible that they can move to a different part of the hospital where the resources and staff are not as intensely needed. As described in the above article, a particular patient may not necessarily need to be in the ICU and could be using resources that another may need. This is especially important with Covid-19 as we see how ICUs and hospitals in general are sometimes over capacity and out of resources.

This data was published in January 2020 and is from MIT’s GOSSIS (Global Open Source Severity of Illness Score) initiative. This dataset includes more than 91,000 hospital ICU visits from Argentina, Australia, New Zealand, Sri Lanka, Brazil, and more than 200 hospitals in the United States (https://physionet.org/content/widsdatathon2020/1.0.0/).  Our chosen dataset focuses on determining patient outcomes based on data taken during their first 24 hours in the ICU. The response variable, hospital_death, is binary with a value of 1 indicating a patient’s death and a value of 0 corresponding to a patient’s survival. 

Some of the data that is gathered pertains to a person’s identity, demographics, vitals and labs from when they enter the ICU, APACHE covariates, APACHE comorbidities, and APACHE predictions and groupings. APACHE refers to Acute Physiology And Chronic Health Evaluation, and it is meant to help with benchmarking especially for accurately predicting mortality. 

After evaluation, the random forest model was shown to be more effective than all other evaluated models, including the APACHE model, at predicting patient mortality. The model achieved an accuracy of over 92% with an F1 score of 0.59 and an AUC of 0.92. 

## Exploratory Data Analysis

To begin the exploration, the response variable, hospital_death, was examined. The outcome variable is hospital_death, which is binary with a value of 1 indicating a patient’s death and a value of 0 indicating a patient’s survival. Distribution of the count for each outcome variable shown in Figure 1 below:

![Figure1](https://github.com/q-maze/sparkHospital/blob/871b97d37693bfb255132aed968c9641b7d6eaf9/images/OutcomeEDA.png)

As shown in the figure above, our response feature is moderately imbalanced, with an approximate ratio of 10:1. This imbalance motivated exploration of imbalance compensation options during the model building process.

Next, a new dataframe is created using a groupBy to aggregate on hospital_id and provide the death percentage per hospital. It is clear that the distribution of hospital death percentages is very left skewed, which is advantageous for patients. 


![Figure2](https://github.com/q-maze/sparkHospital/blob/871b97d37693bfb255132aed968c9641b7d6eaf9/images/hospitalByHospitalDeath.png)

Several categorical features in the dataset including the ICU type, which body system was determined as the cause of admission for use in the APACHE scores, if the stay was the result of an elective surgery, and distribution of patients by ethnicity and gender were also explored.

![Figure3a](https://github.com/q-maze/sparkHospital/blob/871b97d37693bfb255132aed968c9641b7d6eaf9/images/EDA1.png)

It is clear from this analysis that Med/Surg ICUs are the most common type in the dataset and that cardiovascular issues are by far the most commonly evaluated item in the APACHE indexes.


![Figure3b](https://github.com/q-maze/sparkHospital/blob/871b97d37693bfb255132aed968c9641b7d6eaf9/images/EDA2.png)

Exploratory analysis was also performed on hospital_admit_source, icu_admit_source, and icu_stay_type. The hospital_admit_source refers to the location of the patient prior to being admitted to the hospital, and the highest count is the Emergency Department, which is around 35,000 with the next location being much less frequent at less than 10,000 at the operating room. The most frequent icu_admit_source is the accident and emergency unit, and the most frequent icu_stay_type is admit, which refers to the type of unit admission for the patient. 

![Figure3c](https://github.com/q-maze/sparkHospital/blob/871b97d37693bfb255132aed968c9641b7d6eaf9/images/EDA3.png)

## Methods

### Data Preprocessing Pipeline
First, any columns that had more than 50% missing data were removed because with all of the missing rows, imputing these and performing feature engineering may affect the distribution of the data and may not accurately represent the dataset. “Identifier” variables were also dropped, as they are simply IDs that would not contribute to a person’s mortality. Demographic variables such as age, body mass index (BMI), and gender, were kept as they could help shape an accurate prediction. It was also decided to use only features apache_hospital_death_prob and apache_icu_death_prob rather than all of the APACHE covariates because these covariates are used to make up those two probabilities. All APACHE comorbidity and APACHE grouping columns were also kept,  as those are separate from the covariates and are not associated with the probabilities. Based on the data dictionary, the columns labeled in Category indicating "noninvasive" were removed as they are included in the equivalent measurements, which are measured either invasive or non invasive. Finally, the GOSSIS example prediction was dropped due to the APACHE hospital and ICU death probabilities being retained for possible use in models.

After import and preprocessing, 80% of the dataset was retained as a training and validation set and the remaining 20% was reserved as a test set.
 
### Feature Engineering
After dropping the indicated variables, other data preprocessing steps were implemented to further clean the dataset. A set of columns that contain the minimum and maximum values for certain labs drawn during the first hour or day in the hospital were converted into a single column containing the mean of the two values. For example, the diastolic blood pressure maximum and diastolic blood pressure minimum value were averaged into one column, and the two original columns were removed from the dataset. In addition, all missing values in the string columns are converted to `None` type so that they could be imputed to the mean of the non-missing data. Finally, all categorical variables were indexed and one-hot encoded, while all continuous variables were imputed to fill any empty values with the mean value.

### Feature Selection 
As the dataset contained over a hundred possible features and the project team had minimal subject matter expertise, manual feature selection would be time consuming and potentially sub-optimal. Therefore, to determine the features used for modeling, two UnivariateFeatureSelector pipelines were constructed; one for categorical features and one for continuous features. Categorical features were passed to a StringIndexer object to convert them to integers and then assembled into a vector to process them by the feature selector pipeline to choose which variables are most useful in predicting the response. As these features are categorical with a binary response, the UnivariateFeatureSelector uses a Chi-Squared test for determining which variables to include in the model. All categorical variables are then put into a final feature vector. The continuous features are also assembled into a vector to pass through the UnivariateFeatureSelector object and imputed for any missing data to the feature mean. In the case of a continuous predictor and a binary response, the UnivariateFeatureSelector uses an ANOVA F-Test to determine which variables to include in the model.


### Model Selection
Each model was performed on a training/validation test split on the data, reserving 80% of the data for training and validation set, and 20% for testing set. Due to imbalanced data between the response variable, we performed logistic regression on the data with the full dataset and a downsampled set to determine if there would be any improvements on the model. Upon analysis of the downsampled model performance against the non-downsampled model performance, we determined that downsampling was detrimental to overall model performance, so our team decided to proceed with model training on the normal dataset and handle response imbalance with probability threshold selection. 

Next, pipelines were constructed to train each model. Each of these pipelines included a 5-fold cross validation that utilized average AUC for selection of model hyperparameters. The selected model was used to determine predictions on the test data, which were then used to evaluate the metrics of accuracy, precision, recall, F1, AUC, as well as produce a confusion matrix.

## Models

### APACHE Logistic Regression Baseline
For the APACHE variables only model, we included the variables within the dataset containing APACHE hospital and icu death probabilities. The data dictionary explains these two variables as probabilistic predictions of mortality for a patient utilizing the APACHE III score and other covariates, including diagnosis. This logistic regression model was explored to determine if the response variable could be determined only using these two APACHE designated variables. This model is considered our base model and did not adjust for class imbalances.

### Logistic Regression
Our logistic regression model utilized the features selected from the data import and preprocessing pipelines. The selected continuous features were scaled as part of this pipeline. We chose not to process the data further to account for any downsampling. Using a 5-fold cross validation, a logistic regression model with an elastic net regularization parameter was constructed. 

### Downsampled Logistic Regression
We also chose to fit a logistic regression model with an elastic net regularization parameter using the same features as the first logistic regression model, but using a downsampled dataset to account for the 10:1 imbalance in hospital_death.

### Ridge/Lasso Regression
Ridge and Lasso regression models were both performed using the subset of variables from the import and preprocessing pipelines. They both used the pyspark.ml.classification LogisticRegression package, but lasso used an elasticNetParam of 1 while ridge used an elasticNetParam of 0. To tune this model, the parameter grid used for the cross validator contained values for lambda as the hyperparameter.  

### Random Forest
As random forest models often perform well at classification tasks, our group also decided to apply this model type to our dataset. The model was trained using the variables selected from our import and preprocessing pipelines and was not downsampled. To tune this model, our cross validator was passed a parameter grid containing values for the number of trees to use, the maximum number of bins, and the maximum depth of each tree. 

### Gradient Boosted Tree
Gradient boosted trees are another model type that often perform as well or better than random forests at classification tasks, therefore our group chose to include this model type in our analysis. This model was trained using the variables selected from our import and preprocessing pipelines and was not downsampled.  As with the random forest, model hyperparameters were chosen by cross validation. The parameter grid for this model included  the maximum number of bins and the max depth hyperparameters.

### Support Vector Machine
Support vector machine modeling was performed using the subset of features that were preprocessed and selected in our feature selector. After training and tuning the model using the LinearSVC function, the model was evaluated using the default parameters with a linear kernel. Because support vectors are determined based on distance of separating hyperplanes they do not output probabilities. This limited our ability to determine the area under the curve using the same metrics as other models, and limited our ability to tune the cut off value for determining the optimal cut off in each support vector model. For future model evaluation, we could include converting the data to float type and an rdd to output a correct area under the curve. For determining probability and tuning cut off values, Platt scaling was suggested as an alternative.

## Discussion & Conclusions
Based on the evaluation of the tested models, we believe that our group has created several robust models for predicting patient mortality during their first 24 hours in the ICU. Two models appear to have better performance at predicting patient mortality: Random Forest and Gradient Boosted Tree. These ensemble models outperform the other logistic regression models for both F1 and AUC scores. These two models offer close to 0.85 AUC, which outperforms the baseline APACHE model. However, the random forest model offers slightly better performance, both in terms of AUC and F1, over the Gradient Boosted Tree, therefore it was chosen as our champion model.

The logistic regression models all performed similarly, which is in line with our team’s expectations due to the fact that they are similar to one another except for the penalty terms. Our team also chose these models to explore negative class downsampling. While downsampling may have improved precision and recall when compared to the non-downsampled logistic regression models, the associated loss in overall model accuracy made the model untenable for use in production due to the adverse effects of both false positives and false negatives on patient outcomes. We posit that the presence of moderate imbalance in our dataset combined with the small overall size of the positive class may have contributed to the poor performance of the downsampled model. This observation motivated our team’s decision to forgo downsampling for all other models. 

Future work in this area for our team will include exploration of additional Feature Engineering, including working with subject matter experts to determine appropriate features to bucketize. Possible examples include bucketizing individuals by BMI into buckets for normal weight, overweight, and obese. Feature engineering could also include missing data that is dropped from our analysis. Perhaps with subject matter expert input we would discover that the absence of these features in the dataset indicates certain uncollected features.

Another possible area for exploration for future work would be data upsampling or weighting to adjust for the effects of the imbalance in our dataset. While downsampling did not provide a boost in overall model performance, it did improve precision and recall when compared to the identical non-downsampled model. Perhaps upsampling or weighting could improve these measures as well without the harmful effects to overall model performance. 

In addition to feature engineering and data sampling, our team would also like to explore larger parameter spaces for hyperparameter tuning of several of the models, especially the random forest and gradient boosted tree model. The random forest model parameter search yielded the maximum values in the parameter space, which may indicate that further increasing the hyperparameters may yield a more robust model. Additional time and computational resources are required to explore this possibility.



