# Feaure Selection Collection
In industry, many times we need to generate features, understanding them and generate more to improve model performance. I'm taking notes of some method that may help do further exploration.

## SHAP
* How to calculate shaply value step by step: https://www.youtube.com/watch?v=fbrVvMU8T6o
  * It originated from game theory. The basic idea is, there are several players share icecream, shaply calculates, if remove one of the players, how much share the rest of players will get
* It's amethod used to deal with the draw back of XGBoost feature selection
* [You know what, SHAP value came from game theory][2]
  * "Shapley values correspond to the contribution of each feature towards pushing the prediction away from the expected value."
* [My practice code 1 - XGBoost Regressor][1]
* Github has blocked the loading of JS, in fact it provides a method to interact with each record and understand how the feature values affect the prediction
![shap JS](https://github.com/hanhanwu/Hanhan_Data_Science_Practice/blob/master/Better4Industry/Feature_Selection_Collection/xgboost_shap.PNG)

  * In this force plot, the "base value" of the estimated average predicted value from the model training (in new SHAP version, it's using leaf nodes and no longer the same as avg of forecasted values...), the "output value" is the predicted value of current observation. Pink is the position impact (dragging the prediction value higher or towards 1) while blue is the negative impact (dragging the prediction value lower or towards 0).
  * The length of the bar for each feature indicates to which extent the feature affect the forecasted value
  * `output value = base value + sum(all features' SHAP values)`
    * Because of this, sometimes when you got negative forecast values, you can shift the output value to the right and split the shifted difference to each feature. By doing this the base value will stay the same, feature's impact visually stay almost the same and the forecasted value has been "corrected".
      * BTW, [here's an explaination][7] about why you may get negative forecasting values in a boosting regressor even when the training target values are all positive
* [SHAP decision plot][3]
  * [Code example][4]
    * Just need trained model and feature values, no need lables in the data
    * Add `link='logit'` in the `decision_plot()` function will convert log odds back to prediction probabilities (SHAP value is log odds)
  * [SHAP decision source code][5]
    * In the `decision plot`, by default the features are ordered by importance order. When there are multiple records, the importance of a feature is the sum of absolute shap values of the feature
  * Include multiple records plot
  * For each record, it shows how does each feature value lead to the final prediction result

## Tips
* The base value generated from TreeExplainer `expected_value` can be different from the average forecatsed result when using model `predict()`, when [the TreeExplainer depends on some settings from the training data, such as leaf sample weights for random subsample][8]
  * With SHAP version <=0.33, you can pass X_train to SHAP instead of using the trained model
  * Otherwise, the new version of SHAP is no longer the "average forecasted value", it's calcukated using leaf nodes
* When the data is large, you can use `clustered_df = shap_kmeans(df)` and put this clustered_df in a shap explainer. This method helps speed up the computation
  * In SHAP, for each feature subset (2^m - 2) it perturbs the values of features and makes prediction to see how peturbing a feature subset changes the prediction of model. For each feature subset (e.g. [0,1,1,0,0,0] only perturbing feature 2nd and 3rd) you can replace the feature values by any of the values in the training set. By default it does that exhaustively for all points in training, therefore the total number of model predictions it evaluates is n2^m. <b>So, we use shap.kmeans to only perturb based on some representatives (10 centroids instead of 1000 datapoints)</b>
    * `m` is the feature number
    * `N` is the number of samples
    * Total number of model evaluation is `N * (2^m - 2)` 
    * Although the value is randomly assigned for the perturbed values, it's choosing the possible value from the feature that appeared in the training data 
  * There are [different types of SHAP explainer][6]
    * `KernelExplainer` is generic and can be used for all types of models, but slow. 
      * That's also why when using TreeExplainer, you don't have to use `shap.kmeans` for large dataset, since it's fast 
      * KernelExplainer is not applicable for more than 15 features
* When doing the experiments of SHAP performance, there are multiple things can check 
  * Time efficiency for different number of samples, differnt number of features, different model sizes (such as different tree numbers)
  * While the time efficiency has been improved, how's the accuracy of model predictions 
* I don't think SamplingExplainer can replace TreeExplainer for ensembling models
  * It requires a model input from `train()`, and cannot use loaded trained model
  * When there are 3000+ samples, TreeExplainer is still faster  
* Display shap plots on Databricks Notebooks
  * Set `matplotlib=True` so that you don't need to initiate JS. But on Databricks, this is not supported in Force Plot for multiple records now...

```
import matplotlib.pyplot as plt

exp_shap = shap.TreeExplainer(model)
shap_tree = exp_shap.shap_values(X_test)
expected_tree = exp_shap.expected_value
if isinstance(expected_tree, list):
  expected_tree = expected_tree[1]
print(f"Explainer expected value (Base Value): {expected_tree}")

idx = 10

print(f'Force Plot for #"{idx}" observation in test dataframe:')
## Option 1 - hide feature values
shap_force_plot = shap.force_plot(expected_tree, shap_tree[idx], feature_names=X_test_cols, matplotlib=True)
## Option 2 - show feature values
shap_force_plot = shap.force_plot(expected_tree, shap_tree[idx], X_test.iloc[idx], matplotlib=True)
display(shap_force_plot)

print(f'Decision Plot for #"{idx}" observation:')
print('Base Value:', expected_tree)
## Option 1 - hide feature values
shap_deci_plot = shap.decision_plot(expected_tree, shap_tree[idx], X_test.iloc[idx])
## Option 2 - show feature values
shap_deci_plot = shap.decision_plot(expected_tree, shap_tree[idx], feature_names=list(X_test.columns))
display(shap_deci_plot)
```


[1]:https://github.com/hanhanwu/Hanhan_Data_Science_Practice/blob/master/Better4Industry/Feature_Selection_Collection/try_shap_xgboost.ipynb
[2]:https://www.analyticsvidhya.com/blog/2019/11/shapley-value-machine-learning-interpretability-game-theory/?utm_source=feedburner&utm_medium=email&utm_campaign=Feed%3A+AnalyticsVidhya+%28Analytics+Vidhya%29
[3]:https://towardsdatascience.com/introducing-shap-decision-plots-52ed3b4a1cba
[4]:https://slundberg.github.io/shap/notebooks/plots/decision_plot.html
[5]:https://github.com/slundberg/shap/blob/6af9e1008702fb0fab939bf2154bbf93dfe84a16/shap/plots/_decision.py#L46
[6]:https://shap-lrjball.readthedocs.io/en/docs_update/api.html
[7]:https://datascience.stackexchange.com/questions/565/why-does-gradient-boosting-regression-predict-negative-values-when-there-are-no
[8]:https://github.com/slundberg/shap/issues/318
