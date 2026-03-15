# Gen Z Loyalty Analysis — American Airlines
## BUS 480 Project Documentation

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Data Overview](#2-data-overview)
3. [Data Processing and Cleaning](#3-data-processing-and-cleaning)
4. [Data Overview After Cleaning](#4-data-overview-and-summary-after-cleaning)
5. [Exploratory Data Analysis (EDA)](#5-exploratory-data-analysis-eda)
6. [Machine Learning Methods — Problem 1](#6-machine-learning-methods--problem-1)
7. [Machine Learning Methods — Problem 2](#7-machine-learning-methods--problem-2)
8. [Results and Insights](#8-results-and-insights)

---

## 1. Executive Summary

This project investigates two core business questions for American Airlines:

**Problem 1 — Who is Gen Z compared to other generations?**
How does Gen Z differ from Millennials and Gen X in loyalty behavior? We answer this through exploratory data analysis (EDA) and K-Means clustering, comparing loyalty enrollment rates, engagement levels, and booking patterns across generational cohorts.

**Problem 2 — What motivates Gen Z loyalty?**
What actually makes Gen Z stay loyal? We answer this using two gradient boosting models (XGBoost and LightGBM) paired with SHAP explainability to identify and rank the specific service factors that drive Gen Z loyalty enrollment.

The analysis uses an airline passenger satisfaction dataset with 129,880 total records across 25 columns, covering demographic information, 14 in-flight service ratings, flight details, and satisfaction outcomes.

---

## 2. Data Overview

### 2.1 Data Sources

| Dataset | Rows | Columns | File Size |
|---------|------|---------|-----------|
| `train.csv` | 103,904 | 25 | 12 MB |
| `test.csv` | 25,976 | 25 | 2.9 MB |
| **Combined** | **129,880** | **25** | **~15 MB** |

Both datasets share identical column structures and were combined for the full analysis.

### 2.2 Column Descriptions

| Column | Type | Description |
|--------|------|-------------|
| `id` | Integer | Unique passenger identifier |
| `Gender` | Categorical | Male or Female |
| `Customer Type` | Categorical | "Loyal Customer" or "disloyal Customer" |
| `Age` | Integer | Passenger age (7–85 range) |
| `Type of Travel` | Categorical | "Business travel" or "Personal Travel" |
| `Class` | Categorical | "Business", "Eco", or "Eco Plus" |
| `Flight Distance` | Integer | Distance of the flight in miles |
| `Departure Delay in Minutes` | Float | Minutes delayed at departure |
| `Arrival Delay in Minutes` | Float | Minutes delayed at arrival |
| `satisfaction` | Categorical | **Target**: "satisfied" or "neutral or dissatisfied" |

**14 Service Rating Columns** (rated 0–5 scale):

| Service Dimension | What It Measures |
|-------------------|------------------|
| Inflight wifi service | Quality of in-flight Wi-Fi connectivity |
| Departure/Arrival time convenient | Convenience of scheduled flight times |
| Ease of Online booking | Ease of completing an online booking |
| Gate location | Convenience of gate assignment |
| Food and drink | Quality of in-flight food and beverages |
| Online boarding | Smoothness of online boarding process |
| Seat comfort | Physical comfort of seating |
| Inflight entertainment | Quality of entertainment options |
| On-board service | Quality of crew service during flight |
| Leg room service | Adequacy of legroom space |
| Baggage handling | Efficiency of baggage handling |
| Checkin service | Quality of check-in experience |
| Inflight service | Overall in-flight service quality |
| Cleanliness | Cleanliness of the aircraft |

Rating scale: 0 = no service / not applicable, 1 = very dissatisfied, 5 = very satisfied.

---

## 3. Data Processing and Cleaning

### 3.1 Missing Value Treatment

Only one column had missing values:

| Column | Missing Count | Imputation Method |
|--------|--------------|-------------------|
| `Arrival Delay in Minutes` | ~310 rows (in train set) | Median imputation |

Median was chosen over mean because delay distributions are heavily right-skewed (most flights have 0 delay, a few have very large delays). Median is robust to these outliers.

### 3.2 Column Removal

The unnamed index column and `id` column were dropped as they carry no analytical value.

### 3.3 Feature Engineering

Three new features were created from existing columns:

**Generation Labels** — Derived from `Age` using standard generational cutoffs (2024 reference year):

| Generation | Age Range | Birth Year Range |
|------------|-----------|------------------|
| Gen Z | ≤ 27 | ~1997–2012 |
| Millennial | 28–43 | ~1981–1996 |
| Gen X | 44–59 | ~1965–1980 |
| Boomer | 60+ | Before 1965 |

**Engagement Score** — The mean of all 14 service rating columns for each passenger. This serves as a composite proxy for how engaged and satisfied a member is with the airline's service touchpoints.

**Binary Flags**:
- `is_loyal`: 1 if `Customer Type` == "Loyal Customer", 0 otherwise. This is the **target variable** for Problem 2.
- `is_satisfied`: 1 if `satisfaction` == "satisfied", 0 otherwise.

### 3.4 Categorical Encoding (for ML Models)

For the gradient boosting models in Problem 2, three categorical columns were label-encoded:
- `Class` → `class_encoded`
- `Type of Travel` → `travel_type_encoded`
- `Gender` → `gender_encoded`

Label encoding is appropriate here because XGBoost and LightGBM handle encoded categoricals natively via their tree-splitting mechanism. No one-hot encoding is needed.

---

## 4. Data Overview and Summary After Cleaning

### 4.1 Final Dataset Shape

After combining train and test sets, imputing missing values, and engineering features, the final dataset contains:
- **129,880 rows** (passengers)
- **0 missing values**
- **25 original columns + 5 engineered columns** (`Generation`, `is_loyal`, `engagement_score`, `is_satisfied`, `cluster`)

### 4.2 Generational Breakdown

The dataset contains passengers across all four generations. The exact counts and proportions are produced in Section 2 of the notebook. The generation column enables all downstream generational comparisons.

### 4.3 Target Variable Distribution

- **`is_loyal`** (Problem 2 target): Binary split between "Loyal Customer" and "disloyal Customer". The dataset is expected to be imbalanced, with loyal customers making up a larger share (~82% based on typical airline datasets). Stratified train-test splits are used to preserve this ratio.
- **`satisfaction`** (original target): Approximately balanced between "satisfied" and "neutral or dissatisfied."

### 4.4 Feature Summary

| Feature Category | Count | Range/Scale |
|-----------------|-------|-------------|
| Service ratings | 14 | 0–5 integer scale |
| Delay features | 2 | 0–1600+ minutes (continuous) |
| Flight distance | 1 | 31–4983 miles (continuous) |
| Age | 1 | 7–85 years (continuous) |
| Encoded categoricals | 3 | Label-encoded integers |
| **Total model features** | **21** | |

---

## 5. Exploratory Data Analysis (EDA)

### 5.1 Purpose

EDA directly answers Problem 1 — the generational comparisons are descriptive analysis. No ML model is needed. The goal is to summarize, visualize, and statistically test whether Gen Z differs meaningfully from other generations across loyalty, engagement, and service preferences.

### 5.2 Analyses Performed

**Loyalty Enrollment Rate by Generation**
Bar chart showing the percentage of each generation enrolled as "Loyal Customer." This is the most direct answer to "how does Gen Z differ in loyalty behavior?"

**Engagement Score Distribution by Generation**
Boxplots and violin plots showing the spread of the composite engagement score (mean service rating) by generation. This reveals whether Gen Z rates services lower overall, has more variance, or clusters at certain levels.

**Satisfaction Rate by Generation**
Bar chart comparing the proportion of "satisfied" passengers across generations.

**Generation Comparison Table**
A single summary table aggregating count, loyalty rate, satisfaction rate, mean/median engagement, average flight distance, and average delays for each generation. This is the key deliverable table for Problem 1.

**Service Ratings Heatmap by Generation**
A 4-row (generation) x 14-column (service) heatmap showing mean ratings. This identifies which specific services Gen Z rates differently — for example, if Gen Z rates Wi-Fi higher but food lower, that informs targeted investment.

**Travel Class and Type Distribution**
Stacked bar charts showing how each generation distributes across travel classes (Business, Eco, Eco Plus) and travel types (Business, Personal). This contextualizes loyalty differences — if Gen Z flies Eco more often, their lower engagement may reflect class-of-service effects rather than generational preferences.

**Correlation Heatmap**
A triangular heatmap of all 14 service ratings plus flight distance, delays, loyalty, satisfaction, and engagement score. This reveals multicollinearity between features and identifies which service dimensions correlate most strongly with loyalty and satisfaction.

### 5.3 Statistical Tests

Three hypothesis tests confirm whether observed generational differences are statistically significant or could be due to chance:

| Test | Null Hypothesis | Method |
|------|----------------|--------|
| Loyalty vs Generation | Loyalty enrollment is independent of generation | Chi-square test of independence |
| Engagement vs Generation | Mean engagement score is equal across all generations | One-way ANOVA (F-test) |
| Satisfaction vs Generation | Satisfaction is independent of generation | Chi-square test of independence |

All tests use alpha = 0.05. If p-value < 0.05, the generational difference is statistically significant.

### 5.4 Tools Used

`pandas` for data aggregation and summary statistics, `matplotlib` and `seaborn` for visualization, `scipy.stats` for chi-square and ANOVA tests.

---

## 6. Machine Learning Methods — Problem 1

### 6.1 K-Means Clustering

#### Introduction

K-Means is an unsupervised machine learning algorithm that groups data points into K clusters based on behavioral similarity. It works by:
1. Randomly initializing K cluster centers
2. Assigning each data point to its nearest center (Euclidean distance)
3. Recomputing centers as the mean of assigned points
4. Repeating steps 2–3 until convergence

No labels are needed — the algorithm discovers natural structure in the data on its own.

#### Why K-Means Is Important for This Problem

Generation is a demographic label, not a behavioral identity. Two Gen Z passengers can behave completely differently — one may be a high-engagement business traveler, another a price-sensitive occasional flyer. K-Means finds the actual behavioral segments that cut across generational lines.

By examining which generation dominates each cluster, we learn whether generational labels align with behavioral reality or whether demographic assumptions are misleading. This is critical for American Airlines because marketing campaigns should target behavioral personas, not just age groups.

#### Implementation Details

**Input features (18 total)**: All 14 service ratings + Flight Distance + Departure Delay + Arrival Delay + Age.

**Preprocessing**: StandardScaler standardization (zero mean, unit variance). This is critical because K-Means uses Euclidean distance — without scaling, features with larger numeric ranges (e.g., Flight Distance in thousands) would dominate features with smaller ranges (e.g., service ratings 0–5).

**Cluster selection**: The elbow method (plotting inertia vs K) and silhouette score (measuring cluster cohesion and separation) were used to determine the optimal number of clusters. K values from 2 to 8 were evaluated.

**Final model**: K=4 clusters, fit on the full standardized dataset (129,880 passengers).

#### Results and Output

The K-Means model produces:

- **Elbow chart**: Visual confirmation that K=4 captures most of the variance without over-segmenting
- **Silhouette scores**: Quantitative validation of cluster quality for each K
- **Cluster behavioral profiles**: Mean values of all features within each cluster, revealing distinct personas (e.g., high-engagement frequent flyers vs. disengaged economy travelers)
- **Cluster center heatmap**: Standardized feature values per cluster, showing which services define each segment
- **Generation composition**: Crosstab showing what percentage of each cluster belongs to each generation, and vice versa
- **Persona summary table**: Loyalty rate, engagement score, satisfaction rate, and size for each cluster

**Expected outcome**: We expect 3–5 distinct behavioral segments. Gen Z may concentrate in clusters characterized by lower engagement or different service priorities, but the clusters should reveal that behavioral differences within a generation can be larger than differences between generations.

---

## 7. Machine Learning Methods — Problem 2

### 7.1 XGBoost (Extreme Gradient Boosting)

#### Introduction

XGBoost is a supervised gradient boosting algorithm that builds an ensemble of decision trees sequentially. Each tree corrects the errors of the ones before it — this error-correction chain is the "boosting" mechanism. The process works step by step:

1. Start with a baseline prediction (mean loyalty rate for all Gen Z members)
2. Calculate residuals — how wrong was each prediction?
3. Grow a decision tree on those residuals — which features (Wi-Fi score, seat comfort, flight distance) best explain the errors?
4. Multiply the tree's output by a small learning rate and add it to the running prediction
5. Repeat 100–300 times, each tree attacking remaining errors
6. Early stopping halts training when validation loss stops improving
7. Extract feature importance — Gain (how much each feature reduced prediction error) ranks the loyalty drivers

#### Why XGBoost Is Important for This Problem

The relationship between loyalty drivers and outcomes is **non-linear**. A small improvement in Wi-Fi quality may do nothing until a threshold is crossed, then drive a large enrollment jump. XGBoost handles this natively through its tree-based architecture.

It also handles **mixed data types** (Likert-scale survey scores, continuous distances, binary flags) without manual feature engineering. The built-in **L1 and L2 regularization** prevents overfitting, which is important when the Gen Z subset may be smaller than the full dataset.

Most importantly, XGBoost produces **feature importance rankings** that directly answer "what makes Gen Z stay loyal?" — both enrollment probability AND driver ranking in one model.

#### Implementation Details

**Target variable**: `is_loyal` (binary: 1 = Loyal Customer, 0 = disloyal Customer)

**Feature set (21 features)**: 14 service ratings + Flight Distance + Departure Delay + Arrival Delay + Age + class_encoded + travel_type_encoded + gender_encoded

**Data split**: Gen Z subset only, 80% train / 20% test, stratified by target variable

**Hyperparameters**:
- 300 trees (n_estimators) with early stopping at 20 rounds of no improvement
- Max depth 5 (prevents overly complex individual trees)
- Learning rate 0.1 (standard balance between speed and accuracy)
- Subsample 0.8 and colsample_bytree 0.8 (stochastic regularization)
- L1 regularization (reg_alpha) = 1.0 and L2 regularization (reg_lambda) = 1.0

**Validation**: 5-fold stratified cross-validation on the full Gen Z dataset

#### Results and Output

- **ROC-AUC score**: Measures the model's ability to distinguish loyal from non-loyal Gen Z members (0.5 = random, 1.0 = perfect). A strong model should achieve ROC-AUC > 0.70.
- **Classification report**: Precision, recall, and F1-score for both classes (Loyal and Not Loyal)
- **Cross-validation ROC-AUC**: Mean and standard deviation across 5 folds, confirming the model generalizes and isn't overfitting to a single split
- **Feature importance chart (Gain)**: Horizontal bar chart ranking all 21 features by how much they contributed to reducing prediction error. The top 3–5 features are the key loyalty drivers for Gen Z.

**Expectation**: We expect service-quality features (Wi-Fi, online boarding, seat comfort) to rank higher than operational features (delays, flight distance) as loyalty drivers, since service experience is more closely tied to brand loyalty decisions.

---

### 7.2 LightGBM (Light Gradient Boosting Machine)

#### Introduction

LightGBM is Microsoft's implementation of gradient boosting. It uses the same core concept as XGBoost — sequential trees correcting each other's errors — but differs in two key ways:

1. **Leaf-wise growth**: Instead of growing trees level-by-level (balanced), LightGBM finds the single leaf that reduces error the most and splits that one first, regardless of tree balance. This produces more accurate trees with fewer splits.
2. **Histogram-based binning**: Continuous features are binned into discrete histograms before splitting, dramatically reducing computation time (typically 3–10x faster than XGBoost on large datasets).

#### Why Running Both Models Is Important

Running two models is standard ML practice for validation:

- **Model agreement = confidence**. If both XGBoost and LightGBM rank "Online boarding" as the #1 loyalty driver, that finding is robust and can be reported with high confidence.
- **Model disagreement = insight**. If XGBoost flags "Inflight entertainment" as a top-3 driver but LightGBM doesn't, it usually reveals a segment-specific effect worth investigating — perhaps entertainment matters only for long-haul Gen Z travelers.

The Spearman rank correlation between the two models' feature rankings quantifies this agreement (> 0.7 = strong agreement).

#### Implementation Details

Same target variable, feature set, data split, and validation approach as XGBoost.

**LightGBM-specific hyperparameters**:
- num_leaves = 31 (controls tree complexity in leaf-wise growth)
- All other hyperparameters match XGBoost for fair comparison

**Early stopping**: 20 rounds via LightGBM callback functions

#### Results and Output

Same metrics as XGBoost: ROC-AUC, classification report, cross-validation scores, and feature importance chart (using split count rather than gain).

Additionally, a **model comparison table** presents both models side-by-side with ROC-AUC, accuracy, F1-score, and CV scores, and a **feature ranking comparison** table shows XGBoost rank vs LightGBM rank for every feature, flagging agreements and divergences.

---

### 7.3 SHAP Values (SHapley Additive exPlanations)

#### Introduction

SHAP is rooted in cooperative game theory (Shapley values). Standard feature importance tells you what mattered **globally** across all predictions. SHAP tells you **how much each feature contributed to a specific member's prediction and in which direction**.

For example, standard importance might say "Online boarding is important." SHAP says "for this specific Gen Z member, their low online boarding score of 2 **reduced** their loyalty probability by 12 percentage points, while their high seat comfort score of 5 **increased** it by 8 points."

#### Why SHAP Is Important for This Problem

SHAP transforms model outputs into **actionable business recommendations**:

- **High positive SHAP on a service feature** → Invest in that service area (e.g., if high online boarding scores strongly push Gen Z toward loyalty, improve the online boarding experience)
- **High negative SHAP on a feature** → Address the pain point (e.g., if low Wi-Fi scores strongly push Gen Z away from loyalty, prioritize Wi-Fi upgrades)
- **High SHAP variance on a feature** → The feature matters a lot for some Gen Z members but not others, suggesting the need for segmented campaigns rather than one-size-fits-all approaches
- **SHAP interactions** → When two features compound each other (e.g., members who rate both online boarding AND inflight entertainment highly show disproportionately higher loyalty), pair those improvements together

#### Implementation Details

- **TreeExplainer**: SHAP's optimized explainer for tree-based models, used for both XGBoost and LightGBM
- **Scope**: SHAP values computed on the Gen Z test set

#### Results and Output

1. **SHAP Summary Plot (Beeswarm)**: Every dot is one Gen Z member's SHAP value for one feature. Color shows the feature value (red = high, blue = low). Position shows impact on prediction. This is the most information-dense visualization in the analysis.

2. **SHAP Bar Plot**: Mean absolute SHAP values per feature — a cleaned-up global ranking of feature impact.

3. **SHAP Dependence Plots**: For the top 4 features, scatter plots showing feature value (x-axis) vs SHAP contribution (y-axis). These reveal non-linear thresholds — e.g., "Wi-Fi rating below 3 has no effect, but above 3 each point increase adds significant loyalty lift."

4. **SHAP Waterfall Plots**: Individual member explanations showing exactly how each feature pushed the prediction from the base rate to the final probability. Two examples are shown — one member predicted as loyal and one predicted as not loyal — to illustrate contrasting driver profiles.

**Expected findings to look for**:
- Which service features have the highest mean |SHAP| (these are the primary loyalty drivers)
- Whether the direction is consistent (always positive or mixed) — mixed directions suggest segment-specific effects
- Whether threshold effects exist in the dependence plots (e.g., a "tipping point" rating value)

---

## 8. Results and Insights

### 8.1 Problem 1 — Who Is Gen Z?

**Loyalty Enrollment**: The generational comparison table and bar charts reveal how Gen Z's loyalty enrollment rate compares to Millennials, Gen X, and Boomers. Statistical testing (chi-square) confirms whether this difference is significant.

**Engagement Levels**: Boxplots and violin plots show the distribution of the composite engagement score by generation. Key observations include whether Gen Z has a lower median engagement, wider spread (more heterogeneous), or a bimodal distribution suggesting two distinct sub-segments.

**Service Preferences**: The generation x service heatmap identifies which of the 14 service dimensions Gen Z rates differently. This directly informs where American Airlines should invest to improve Gen Z satisfaction.

**Travel Behavior**: Class and travel type distributions contextualize loyalty differences — Gen Z's lower loyalty may partly reflect that they fly economy and for personal travel more often, not necessarily that they dislike the airline.

**Behavioral Clusters**: K-Means identifies 4 behavioral personas that cut across generational lines. The generation composition of each cluster reveals whether Gen Z concentrates in specific behavioral segments (e.g., over-represented in a "disengaged economy" cluster) or distributes evenly. This is critical because it determines whether generational targeting or behavioral targeting is more effective for marketing.

### 8.2 Problem 2 — What Motivates Gen Z Loyalty?

**Model Performance**: Both XGBoost and LightGBM are evaluated on ROC-AUC, accuracy, F1-score, and 5-fold cross-validation. The performance comparison table shows whether the models achieve reliable predictive power on the Gen Z subset.

**Top Loyalty Drivers**: The feature importance rankings from both models are compared. Features where both models agree on high importance are the most confident loyalty drivers. The Spearman rank correlation quantifies overall model agreement.

**SHAP Insights**: The SHAP summary plot reveals not just which features matter, but the direction and magnitude of their impact. The dependence plots expose non-linear relationships and threshold effects. The waterfall plots provide concrete, member-level examples that illustrate how the drivers work in practice.

**Actionable Outputs**:
- A ranked list of Gen Z loyalty drivers (agreed upon by both models)
- Enrollment probability scores for every Gen Z member in the test set
- Specific service areas where investment would most impact Gen Z loyalty
- Evidence for whether one-size-fits-all or segmented campaigns are appropriate (based on SHAP variance)

### 8.3 Method Justification Summary

| Method | Problem | Why Chosen |
|--------|---------|------------|
| EDA | 1 | Descriptive comparisons are the direct answer — no model needed for "how do generations differ" |
| K-Means | 1 | Finds behavioral segments that may be more meaningful than demographic labels |
| XGBoost | 2 | Handles non-linear relationships, mixed feature types, and produces feature importance rankings |
| LightGBM | 2 | Validates XGBoost findings through model agreement; leaf-wise growth may capture different patterns |
| SHAP | 2 | Moves from "what matters" to "how much and in which direction for each individual" — enables actionable recommendations |

---

*Analysis notebook: `gen-z-loyalty-analysis.ipynb`*
*Data files: `train.csv`, `test.csv`*
