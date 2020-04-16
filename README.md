Objective: Develope a Fraud Detection model using a synthetic dataset generated by the PaySim mobile money simulator

Link: https://www.kaggle.com/ntnu-testimon/paysim1

Methodology:

  A. Exploratory Data Analysis: 01. Data Exploration.ipynb
  
      Findings:
      1: Frauds are evenly distributed across weeks (Given that each Step as a hour, there is data of 31 days split into 4 sections of 186 hours. Each section is termed as a week)
      2: No missing value in the data
      3: Not many customers repeating transactions in two consequetive weeks. Me vs Me features will be sparse
      4: Negligible fraud among repeat transacting customers
      5: Only CASH_OUT and TRANSFER type of transactions are flagged Fraud
      6: isFlaggedFraud is only 1 when isFraud is 1
      7: All transactions above the minimum amount when isFlaggedFraud==1 is not Fraud
      8: isFlaggedFraud definition is not consistent with the definition provided in the variable description
      9: For all isFlaggedFraud==1, oldBalanceDest=newBalanceDest=0 and transaction type is only TRANSFER. No consistent patterm in other variables when isFlaggedFraud==1
      10: There are other TRANSFERS where isFlaggedFraud==0 and oldBalanceDest=newBalanceDest=0
      11: No particular threshold for amt_bal_ratio (Amount/Old Balance Orig) when isFlaggedFraud==1. Range overlaps when isFlaggedFraud==0
      12: Merchants are only in Destination Name
      13: Only PAYMENT is done towards Merchants and subsequently no Fraud
      14: Only Customer in Transaction Origin and Customer and Merchant in Transaction Destination. All Frauds is happening in Transactions between Customers
      15: New Balance Orig is not equal to zero when Amount > Old Balance Origin due to CASH IN
      16: New Balance Orig is not equal to zero and Amount > Old Balance Origin and type==CASH OUT data does not tally with Balance movements. There is not corresponding CASH IN or Name in Destination for the corresponding Names
      17: For CASH IN Transactions majority cases NEW Balance Origin - OLD Balance Origin == Amount. For the cases its doesnt hold true, NEW Balance Origin == OLD Balance Origin
      18: New Balance Origination==0 is not a distinguisher between Fraud and Non Fraud transactions
      19: All transactions with Amount==0 are Fraud
      20: Distinct difference in pattern of ZERO Origination and Destination Balance for Fraud and Non Fraud Transactions
      21: Old Balance Orig tend to be higher for Fraud than Non Fraud transactions. Customer with higher balances are usually targeted for Fraud
      
      Actions based on the above findings:
      1: Take first Week 2 and 3 as Train and Week 4 as OOT Validation sample [Findings: 1]
      2: No missing value imputation required [Findings: 2]
      3: Features created from single transactions and me vs others.[Findings: 3,4]
      4: Model data on only CASH_OUT and TRANSFER transactions/Create variable with extra weightage to CASH_OUT and TRANSFER transaction types [Findings: 5]
      5: isFlaggedFraud feature can be dropped [6 - 11]
      6: Create a flag for Payments to Merchants if modelling data on all transaction types[12 - 14]
      7: Assign a standard value for Amount/oldBalanceOrig for CASH IN type transactions while developing model on all types of transaction [15-17]
      8: Assign Flag for Amount==0 [19]
      9: Create categories of Old Balance Orig for CASH_OUT & TRANSFER type transactions [21]
      
  B. Feature Engineering for CASH_OUT & TRANSFER type transactions
  
  1:  02.A. Data Exploration and Feature Engineering - Cash Out & Transfer.ipynb
  
      Feature: $ amount of tranaction and number of transaction done by a customer in last 186 hours
      Since the data had lot ~2.8 million rows it was taking sufficient amount of time to create the feature, so it was not added in model development
          
  2:  02.B. Data Exploration and Feature Engineering - Cash Out & Transfer.ipynb
  
      Feature:
      $TransLW : $ amount of transaction done in last week, i.e if a customer is transacting in week 2 it will capture the $ amount of transaction done in week 1
      NumTransLW: Number of transaction done in last week, i.e if a customer is transacting in week 2 it will capture the number of transaction done in week 1
      FraudBefore: Flag 1 if the Destination Name was in a Fraud transaction before
      oldBalanceOrigCat: Divide Old Balance Orig in 3 bands
      amount_median: Median $ amount of a particular oldBalanceOrigCat
      amount_std: SD $ amount of a particular oldBalanceOrigCat
      amt_max: Max $ amount of a particular oldBalanceOrigCat
      amount_deviance_1: (amount - amount_median)/amount_std
      amount_deviance_2: amount - amount_median
      amount_deviance_3: amt_max - amount
      amount_deviance_4: (amt_max - amount)/amt_max
      amt_bal_perc: amount/oldBalanceOrig
      time_of_day: Dividing every 24 steps into 3 categories
      amount_tod_median: Median $ amount of a particular oldBalanceOrigCat and time_of_day
      amount_tod_std: SD $ amount of a particular oldBalanceOrigCat and time_of_day
      amount_tod_max: Max $ amount of a particular oldBalanceOrigCat and time_of_day
      amount_deviance_5: (amount - amount_tod_median)/amount_tod_std
      amount_deviance_6: amount - amount_tod_median
      amount_deviance_7: amount_tod_max - amount
      amount_deviance_8: (amount_tod_max - amount)/amount_tod_max
      diffBalanceOrig: newBalanceOrig + amount - oldBalanceOrig
      diffBalanceDest: oldBalanceDest + amount - newBalanceDest
         
  C. Model Development
  
  1. 03.A.1 Model Development - Fraud During Transaction.ipynb
  
    Model developed considering detecting fraudulent transaction during occurance of transaction.
    Removing following features during model development:
    1. nameOrig
    2. newBalanceOrig
    3. nameDest
    4. newBalanceDest
    5. isFlaggedFraud
    6. diffBalanceOrig
    7. diffBalanceDest
    
    Dependent Varaible: isFraud
    Model Development Period: Week 2 and 3 [Obs: 1697805 Fraud:4043]
    Out of Time Validation set: Week 4 [Obs: 4043 Fraud:2048]
    Classification Model: XGBoost
    
    Split Model Development Period into Traina nd Test 
    5 Fold Cross Validation on Train data to find optimum n_estimator where Test AUPR curve is maximum
    Optimum n_estimator=1296 (cross validation on 1300 estimators, scope to check by increasing n_estimator max limit)
    Since cross validation took sisgnificant amount of time and other hyperparameter tuning provides marginal gain, develop model keeping other hypermparameters constant
    learning_rate =0.05,
    n_estimators=1300,
    max_depth=5,
    min_child_weight=1,
    gamma=0,
    subsample=0.8,
    colsample_bytree=0.8,
    objective= 'binary:logistic',
    nthread=4,
    scale_pos_weight=419,
    seed=27
    
    Result:
    AUPRC Train= 0.9992531495369624
    AUPRC Test= 0.9832524009221997
    AUPRC OOT= 0.990502966074935
    Threshold Score: 0.99983096 [Minimum score where Precision is maximum, in this case max Precision is 1]
    Depending on the objective of the model, to minimize False Poistive (maximise Precision) or minimize False Negative (maximise TPR) we can choose the threshlod score

    Confusion Matrix at Threshold Score and model KPIs:
    
    Train:
                  Predicted
                  0         1
            0  [1185633       0]
    Actual    
            1  [    578    2252]

    Accuracy: 99.95

    Sensitivity/Recall/TPR: 79.58

    Precision: 100.0
    
    Test:
                  Predicted
                  0         1
            0  [508129      0]
    Actual    
            1  [   266    947]

    Accuracy: 99.95

    Sensitivity/Recall/TPR: 78.07

    Precision: 100.0

    OOT:
                  Predicted
                  0         1
            0  [79582     0]
    Actual    
            1  [  527  1521]

    Accuracy: 99.35

    Sensitivity/Recall/TPR: 74.27

    Precision: 100.0
    
    Model File: xgb_model_1.1
  
   
          
     
