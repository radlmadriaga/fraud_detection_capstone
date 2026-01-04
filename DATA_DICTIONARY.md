# Fraud Detection Capstone Project - Data Dictionary

## Dataset Overview
- **Records:** 50,000 financial transactions
- **Time Period:** Full year 2023 (Jan-Dec)
- **Features:** 21 original + 4 engineered = 25 total
- **Target:** Fraud_Label (Binary: 0 = Legitimate, 1 = Fraudulent)
- **Class Distribution:** 68% Legitimate (33,933), 32% Fraudulent (16,067)

---

## Feature Descriptions

### **Identification & Metadata (Excluded from Modeling)**

| Feature | Type | Range | Description | Notes |
|---------|------|-------|-------------|-------|
| `Transaction_ID` | String | N/A | Unique transaction identifier | Primary key, removed before training |
| `User_ID` | String | N/A | Unique user identifier | Removed before training |
| `Timestamp` | DateTime | 2023-01-01 to 2023-12-31 | Transaction timestamp | Removed before training |

---

### **Transaction Features (Numeric)**

| Feature | Type | Range | Description | Key Insight |
|---------|------|-------|-------------|-------------|
| `Transaction_Amount` | Float | $0.32 - $500 | Amount of transaction in USD | Higher amounts = higher fraud risk |
| `Account_Balance` | Float | $0 - $100k | Current account balance before transaction | Low balance = account strain indicator |
| `Avg_Transaction_Amount_7d` | Float | $10 - $5,000 | 7-day rolling average of transaction amounts | Baseline for anomaly detection |
| `Daily_Transaction_Count` | Integer | 1 - 14 | Number of transactions in past 24 hours | Fraud: 9-14 (testing mode), Normal: 2-4 |
| `Card_Age` | Integer (months) | 1 - 236 | How long card has been active | New cards (< 3 mo) have higher fraud risk |

---

### **Security & Behavioral Features**

| Feature | Type | Range | Description | Red Flag |
|---------|------|-------|-------------|----------|
| `Risk_Score` | Float | 0.0 - 1.0 | Pre-computed bank risk score | > 0.7 = high fraud probability |
| `Previous_Fraudulent_Activity` | Binary | 0, 1 | Has this customer been involved in fraud before? | = 1 (Yes) = 7.5× more likely to fraud |
| `IP_Address_Flag` | Binary | 0, 1 | Is login from unusual/new IP address? | = 1 (Yes) = geographic impossibility |
| `Failed_Transaction_Count_7d` | Integer | 0 - 4 | Failed transactions in past 7 days | > 3 = card testing pattern |

---

### **Merchant & Transaction Type Features**

| Feature | Type | Categories | Description | Fraud Risk |
|---------|------|-----------|-------------|------------|
| `Transaction_Type` | Categorical | POS, Online, ATM Withdrawal, Bank Transfer | Type of transaction | Online slightly higher |
| `Device_Type` | Categorical | Mobile, Laptop, Tablet | Device used for transaction | All similar risk |
| `Location` | Categorical | Sydney, London, Mumbai, New York, Tokyo | Geographic location | All similar risk (audit confirmed) |
| `Merchant_Category` | Categorical | Travel, Clothing, Electronics, Restaurants, Groceries | Merchant business type | Travel slightly higher |
| `Card_Type` | Categorical | Visa, Mastercard, Amex, Discover | Credit card brand | All similar risk |
| `Authentication_Method` | Categorical | Biometric, Password, OTP, PIN | Authentication type used | Biometric = lower risk |

---

### **Target Variable**

| Feature | Type | Values | Distribution | Notes |
|---------|------|--------|---------------|-------|
| `Fraud_Label` | Binary | 0 = Legitimate, 1 = Fraudulent | 68% - 32% | Imbalanced (handled with class_weight='balanced') |

---

## **ENGINEERED FEATURES** (Created during preprocessing)

### 1. **Transaction_to_Avg_Ratio**
```
Formula: Transaction_Amount / (Avg_Transaction_Amount_7d + 1)
Purpose: Detect spending spikes
Example: If avg = $100, transaction = $500 → Ratio = 5.0 (high anomaly)
Red Flag: Ratio > 3.0 often indicates fraud
```

### 2. **Balance_to_Amount_Ratio**
```
Formula: Account_Balance / (Transaction_Amount + 1)
Purpose: Identify account strain
Example: Balance = $5,000, Transaction = $100 → Ratio = 50 (safe)
         Balance = $100, Transaction = $1,000 → Ratio = 0.1 (impossible!)
Red Flag: Ratio < 1.0 means insufficient funds available
```

### 3. **Failed_Transaction_Rate**
```
Formula: Failed_Transaction_Count_7d / (Daily_Transaction_Count + 1)
Purpose: Detect card testing behavior
Example: 3 failures, 10 daily transactions → Rate = 0.30 (30% failure)
Red Flag: Rate > 0.3 indicates desperation/fraud attempts
```

### 4. **Composite_Risk**
```
Formula: 0.3×Risk_Score + 0.3×Previous_Fraud + 0.2×IP_Flag + 0.2×(Failed/4)
Purpose: Aggregate multiple risk signals
Example: Risk=0.8, Prev_Fraud=1, IP=1, Failures=3 → Composite = 0.875 (Very High!)
Red Flag: > 0.7 warrants immediate investigation
```

---

## **Data Quality Checks**

### **Missing Values**
- Total missing: **0** (complete dataset)
- Action: None needed
- Benefit: No imputation bias

### **Duplicates**
- Duplicate records: **0** (verified)
- Action: None needed
- Note: Duplicate transactions would be legitimate (same user, different times)

### **Outliers**
- Outliers detected (IQR method): ~2,400 (4.8%)
- Action: **KEPT** (outliers often = fraud signals)
- Rationale: Unusual transactions are fraud indicators

### **Class Imbalance**
- Original ratio: 2.11:1 (Legitimate:Fraudulent)
- Severity: Moderate
- Handling: `class_weight='balanced'` in models

---

## **Feature Preprocessing Applied**

### **Numeric Features**
- [x] Checked for missing values
- [x] Identified outliers (kept for fraud signals)
- [x] Standardized (StandardScaler) for models that require it
- [x] Engineered 4 new features based on domain knowledge
- [x] Selected via correlation filtering (|corr| > 0.01)

### **Categorical Features**
- [x] One-Hot Encoded (drop_first=True to avoid multicollinearity)
- [x] Applied only to modeling features (ID/timestamp excluded)

### **Training Data Pipeline**
1. Drop ID, timestamp, target columns
2. Engineer 4 new features
3. One-hot encode categorical features
4. Correlation-based feature selection
5. Stratified 80/20 train/test split
6. StandardScaler fit on training, transform on test

---

## **Feature Correlations with Fraud (Top 15)**

| Rank | Feature | Correlation | Direction | Strength |
|------|---------|-------------|-----------|----------|
| 1 | Risk_Score | 0.283 | Positive | Strong |
| 2 | Previous_Fraud | 0.182 | Positive | Moderate |
| 3 | Daily_Count | 0.121 | Positive | Weak-Moderate |
| 4 | IP_Address_Flag | 0.098 | Positive | Weak |
| 5 | Failed_Count | 0.086 | Positive | Weak |
| 6 | Trans_to_Avg_Ratio | 0.054 | Positive | Very Weak |
| 7 | Avg_Amount_7d | 0.049 | Positive | Very Weak |
| 8 | Card_Age | 0.037 | Positive | Very Weak |
| 9 | Balance_to_Amount | 0.032 | Positive | Very Weak |
| 10 | Account_Balance | 0.028 | Positive | Very Weak |

---

## **Data Limitations**

1. **Synthetic Data:** Patterns generated, may not fully reflect real-world fraud
2. **Single Year:** No multi-year trends, seasonal patterns
3. **No Temporal:** Transaction sequencing not modeled
4. **Static Features:** No time-series behavioral patterns
5. **Limited Merchant Data:** Only 5 categories, real-world has 1000s

---

## **Special Notes**

- Features with missing values: 0
- Features requiring scaling: All numeric (StandardScaler applied)
- Features requiring encoding: 6 categorical (One-Hot applied)
- Features causing data leakage: None identified
- Features with strong correlations (>0.7): None (good multicollinearity)
- Imbalanced target: Yes, but handled appropriately

---

**Last Updated:** January 4, 2026  
**Preprocessor Version:** 1.0  
**Data Version:** synthetic_fraud_dataset.csv v1