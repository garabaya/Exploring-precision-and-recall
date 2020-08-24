# Exploring precision and recall

The goal of this second notebook is to understand precision-recall in the context of classifiers.

 * Use Amazon review data in its entirety.
 * Train a logistic regression model.
 * Explore various evaluation metrics: accuracy, confusion matrix, precision, recall.
 * Explore how various metrics can be combined to produce a cost of making an error.
 * Explore precision and recall curves.
 
Because we are using the full Amazon review dataset (not a subset of words or reviews), in this assignment we return to using Turi Create for its efficiency. As usual, let's start by **firing up Turi Create**.

Make sure you have the latest version of Turi Create.


```python
from __future__ import division
import turicreate
import numpy as np
```

# Load amazon review dataset


```python
products = turicreate.SFrame('amazon_baby.sframe/')
```

# Extract word counts and sentiments

As in the first assignment of this course, we compute the word counts for individual words and extract positive and negative sentiments from ratings. To summarize, we perform the following:

1. Remove punctuation.
2. Remove reviews with "neutral" sentiment (rating 3).
3. Set reviews with rating 4 or more to be positive and those with 2 or less to be negative.


```python
import string

def remove_punctuation(text):
    try: # python 2.x
        text = text.translate(None, string.punctuation) 
    except: # python 3.x
        translator = text.maketrans('', '', string.punctuation)
        text = text.translate(translator)
    return text

# Remove punctuation.
review_clean = products['review'].apply(remove_punctuation)

# Count words
products['word_count'] = turicreate.text_analytics.count_words(review_clean)

# Drop neutral sentiment reviews.
products = products[products['rating'] != 3]

# Positive sentiment to +1 and negative sentiment to -1
products['sentiment'] = products['rating'].apply(lambda rating : +1 if rating > 3 else -1)
```

Now, let's remember what the dataset looks like by taking a quick peek:


```python
products
```




<div style="max-height:1000px;max-width:1500px;overflow:auto;"><table frame="box" rules="cols">
    <tr>
        <th style="padding-left: 1em; padding-right: 1em; text-align: center">name</th>
        <th style="padding-left: 1em; padding-right: 1em; text-align: center">review</th>
        <th style="padding-left: 1em; padding-right: 1em; text-align: center">rating</th>
        <th style="padding-left: 1em; padding-right: 1em; text-align: center">word_count</th>
        <th style="padding-left: 1em; padding-right: 1em; text-align: center">sentiment</th>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">Planetwise Wipe Pouch</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">it came early and was not<br>disappointed. i love ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">5.0</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">{&#x27;recommend&#x27;: 1.0,<br>&#x27;highly&#x27;: 1.0, ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">Annas Dream Full Quilt<br>with 2 Shams ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">Very soft and comfortable<br>and warmer than it ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">5.0</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">{&#x27;quilt&#x27;: 1.0, &#x27;this&#x27;:<br>1.0, &#x27;for&#x27;: 1.0, ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">Stop Pacifier Sucking<br>without tears with ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">This is a product well<br>worth the purchase.  I ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">5.0</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">{&#x27;tool&#x27;: 1.0, &#x27;clever&#x27;:<br>1.0, &#x27;approach&#x27;: 2.0, ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">Stop Pacifier Sucking<br>without tears with ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">All of my kids have cried<br>non-stop when I tried to ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">5.0</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">{&#x27;rock&#x27;: 1.0,<br>&#x27;headachesthanks&#x27;: 1.0, ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">Stop Pacifier Sucking<br>without tears with ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">When the Binky Fairy came<br>to our house, we didn&#x27;t ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">5.0</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">{&#x27;thumb&#x27;: 1.0, &#x27;or&#x27;: 1.0,<br>&#x27;break&#x27;: 1.0, &#x27;trying&#x27;: ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">A Tale of Baby&#x27;s Days<br>with Peter Rabbit ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">Lovely book, it&#x27;s bound<br>tightly so you may no ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">4.0</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">{&#x27;2995&#x27;: 1.0, &#x27;for&#x27;: 1.0,<br>&#x27;barnes&#x27;: 1.0, &#x27;at&#x27;:  ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">Baby Tracker&amp;reg; - Daily<br>Childcare Journal, ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">Perfect for new parents.<br>We were able to keep ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">5.0</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">{&#x27;right&#x27;: 1.0, &#x27;because&#x27;:<br>1.0, &#x27;questions&#x27;: 1.0, ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">Baby Tracker&amp;reg; - Daily<br>Childcare Journal, ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">A friend of mine pinned<br>this product on Pinte ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">5.0</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">{&#x27;like&#x27;: 1.0, &#x27;and&#x27;: 1.0,<br>&#x27;changes&#x27;: 1.0, &#x27;the&#x27;: ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">Baby Tracker&amp;reg; - Daily<br>Childcare Journal, ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">This has been an easy way<br>for my nanny to record ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">4.0</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">{&#x27;in&#x27;: 1.0, &#x27;pages&#x27;: 1.0,<br>&#x27;out&#x27;: 1.0, &#x27;run&#x27;: 1.0, ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">Baby Tracker&amp;reg; - Daily<br>Childcare Journal, ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">I love this journal and<br>our nanny uses it ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">4.0</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">{&#x27;tracker&#x27;: 1.0, &#x27;now&#x27;:<br>1.0, &#x27;postits&#x27;: 1.0, ...</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
    </tr>
</table>
[166752 rows x 5 columns]<br/>Note: Only the head of the SFrame is printed.<br/>You can use print_rows(num_rows=m, num_columns=n) to print more rows and columns.
</div>



## Split data into training and test sets

We split the data into a 80-20 split where 80% is in the training set and 20% is in the test set.


```python
train_data, test_data = products.random_split(.8, seed=1)
```

## Train a logistic regression classifier

We will now train a logistic regression classifier with **sentiment** as the target and **word_count** as the features. We will set `validation_set=None` to make sure everyone gets exactly the same results.  

Remember, even though we now know how to implement logistic regression, we will use Turi Create for its efficiency at processing this Amazon dataset in its entirety.  The focus of this assignment is instead on the topic of precision and recall.


```python
model = turicreate.logistic_classifier.create(train_data, target='sentiment',
                                              features=['word_count'],
                                              validation_set=None)
```


<pre>Logistic regression:</pre>



<pre>--------------------------------------------------------</pre>



<pre>Number of examples          : 133416</pre>



<pre>Number of classes           : 2</pre>



<pre>Number of feature columns   : 1</pre>



<pre>Number of unpacked features : 121712</pre>



<pre>Number of coefficients      : 121713</pre>



<pre>Starting L-BFGS</pre>



<pre>--------------------------------------------------------</pre>



<pre>+-----------+----------+-----------+--------------+-------------------+</pre>



<pre>| Iteration | Passes   | Step size | Elapsed Time | Training Accuracy |</pre>



<pre>+-----------+----------+-----------+--------------+-------------------+</pre>



<pre>| 0         | 4        | 0.250000  | 1.418637     | 0.840754          |</pre>



<pre>| 1         | 9        | 3.250000  | 1.961042     | 0.941514          |</pre>



<pre>| 2         | 11       | 2.778177  | 2.211037     | 0.942638          |</pre>



<pre>| 3         | 12       | 2.778177  | 2.388471     | 0.967822          |</pre>



<pre>| 4         | 13       | 2.778177  | 2.549495     | 0.976495          |</pre>



<pre>| 5         | 14       | 2.778177  | 2.711991     | 0.976495          |</pre>



<pre>+-----------+----------+-----------+--------------+-------------------+</pre>


# Model Evaluation

We will explore the advanced model evaluation concepts that were discussed in the lectures.

## Accuracy

One performance metric we will use for our more advanced exploration is accuracy, which we have seen many times in past assignments.  Recall that the accuracy is given by

$$
\mbox{accuracy} = \frac{\mbox{# correctly classified data points}}{\mbox{# total data points}}
$$

To obtain the accuracy of our trained models using Turi Create, simply pass the option `metric='accuracy'` to the `evaluate` function. We compute the **accuracy** of our logistic regression model on the **test_data** as follows:


```python
accuracy= model.evaluate(test_data, metric='accuracy')['accuracy']
print("Test Accuracy: %s" % accuracy)
```

    Test Accuracy: 0.9221862251019919


## Baseline: Majority class prediction

Recall from an earlier assignment that we used the **majority class classifier** as a baseline (i.e reference) model for a point of comparison with a more sophisticated classifier. The majority classifier model predicts the majority class for all data points. 

Typically, a good model should beat the majority class classifier. Since the majority class in this dataset is the positive class (i.e., there are more positive than negative reviews), the accuracy of the majority class classifier can be computed as follows:


```python
baseline = len(test_data[test_data['sentiment'] == 1])/len(test_data)
print("Baseline accuracy (majority class classifier): %s" % baseline)
```

    Baseline accuracy (majority class classifier): 0.8427825773938085


**Quiz Question:** Using accuracy as the evaluation metric, was our **logistic regression model** better than the baseline (majority class classifier)? Yes

## Confusion Matrix

The accuracy, while convenient, does not tell the whole story. For a fuller picture, we turn to the **confusion matrix**. In the case of binary classification, the confusion matrix is a 2-by-2 matrix laying out correct and incorrect predictions made in each label as follows:
```
              +---------------------------------------------+
              |                Predicted label              |
              +----------------------+----------------------+
              |          (+1)        |         (-1)         |
+-------+-----+----------------------+----------------------+
| True  |(+1) | # of true positives  | # of false negatives |
| label +-----+----------------------+----------------------+
|       |(-1) | # of false positives | # of true negatives  |
+-------+-----+----------------------+----------------------+
```
To print out the confusion matrix for a classifier, use `metric='confusion_matrix'`:


```python
confusion_matrix = model.evaluate(test_data, metric='confusion_matrix')['confusion_matrix']
confusion_matrix
```




<div style="max-height:1000px;max-width:1500px;overflow:auto;"><table frame="box" rules="cols">
    <tr>
        <th style="padding-left: 1em; padding-right: 1em; text-align: center">target_label</th>
        <th style="padding-left: 1em; padding-right: 1em; text-align: center">predicted_label</th>
        <th style="padding-left: 1em; padding-right: 1em; text-align: center">count</th>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">-1</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">896</td>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">-1</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">-1</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">3543</td>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">-1</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1698</td>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">27199</td>
    </tr>
</table>
[4 rows x 3 columns]<br/>
</div>



**Quiz Question**: How many predicted values in the **test set** are **false positives**? 1698


```python
false_positives = confusion_matrix[(confusion_matrix['target_label'] == -1) 
                                   & (confusion_matrix['predicted_label'] == 1)]['count'][0:1]
print(false_positives)
```

    [1698]


## Computing the cost of mistakes


Put yourself in the shoes of a manufacturer that sells a baby product on Amazon.com and you want to monitor your product's reviews in order to respond to complaints.  Even a few negative reviews may generate a lot of bad publicity about the product. So you don't want to miss any reviews with negative sentiments --- you'd rather put up with false alarms about potentially negative reviews instead of missing negative reviews entirely. In other words, **false positives cost more than false negatives**. (It may be the other way around for other scenarios, but let's stick with the manufacturer's scenario for now.)

Suppose you know the costs involved in each kind of mistake: 
1. \$100 for each false positive.
2. \$1 for each false negative.
3. Correctly classified reviews incur no cost.

**Quiz Question**: Given the stipulation, what is the cost associated with the logistic regression classifier's performance on the **test set**? 170696


```python
false_negatives = confusion_matrix[(confusion_matrix['target_label'] == 1) 
                                   & (confusion_matrix['predicted_label'] == -1)]['count'][0:1]
cost = false_positives * 100 + false_negatives
print(cost)
```

    [170696]


## Precision and Recall

You may not have exact dollar amounts for each kind of mistake. Instead, you may simply prefer to reduce the percentage of false positives to be less than, say, 3.5% of all positive predictions. This is where **precision** comes in:

$$
[\text{precision}] = \frac{[\text{# positive data points with positive predicitions}]}{\text{[# all data points with positive predictions]}} = \frac{[\text{# true positives}]}{[\text{# true positives}] + [\text{# false positives}]}
$$

So to keep the percentage of false positives below 3.5% of positive predictions, we must raise the precision to 96.5% or higher. 

**First**, let us compute the precision of the logistic regression classifier on the **test_data**.


```python
precision = model.evaluate(test_data, metric='precision')['precision']
print("Precision on test data: %s" % precision)
```

    Precision on test data: 0.941239575042392


**Quiz Question**: Out of all reviews in the **test set** that are predicted to be positive, what fraction of them are **false positives**? (Round to the second decimal place e.g. 0.25) 0.06


```python
1-precision
```




    0.058760424957608026




```python
predicted_positives = confusion_matrix[confusion_matrix['predicted_label'] == 1]['count'].sum()
false_positives / predicted_positives
```




    dtype: float
    Rows: 1
    [0.058760424957608054]



**Quiz Question:** Based on what we learned in lecture, if we wanted to reduce this fraction of false positives to be below 3.5%, we would (select one):

- Discard a sufficient number of positive predictions
- Discard a sufficient number of negative predictins
- **Increase threshold for predicting the positive class ($y_{hat} = +1$) <--**
- Decrease threshold for predicting the positive class ($y_{hat} = +1$)



A complementary metric is **recall**, which measures the ratio between the number of true positives and that of (ground-truth) positive reviews:

$$
[\text{recall}] = \frac{[\text{# positive data points with positive predicitions}]}{\text{[# all positive data points]}} = \frac{[\text{# true positives}]}{[\text{# true positives}] + [\text{# false negatives}]}
$$

Let us compute the recall on the **test_data**.


```python
recall = model.evaluate(test_data, metric='recall')['recall']
print("Recall on test data: %s" % recall)
```

    Recall on test data: 0.9681082043068162


**Quiz Question**: What fraction of the positive reviews in the **test_set** were correctly predicted as positive by the classifier? 0.97 (recall value)

**Quiz Question**: What is the recall value for a classifier that predicts **+1** for all data points in the **test_data**? 1


```python
positives = confusion_matrix[confusion_matrix['target_label'] == 1]['count'].sum()
true_positives = confusion_matrix[(confusion_matrix['target_label'] == 1) 
                                   & (confusion_matrix['predicted_label'] == 1)]['count'][0:1]
true_positives / positives
```




    dtype: float
    Rows: 1
    [0.9681082043068162]



# Precision-recall tradeoff

In this part, we will explore the trade-off between precision and recall discussed in the lecture.  We first examine what happens when we use a different threshold value for making class predictions.  We then explore a range of threshold values and plot the associated precision-recall curve.  


## Varying the threshold

False positives are costly in our example, so we may want to be more conservative about making positive predictions. To achieve this, instead of thresholding class probabilities at 0.5, we can choose a higher threshold. 

Write a function called `apply_threshold` that accepts two things
* `probabilities` (an SArray of probability values)
* `threshold` (a float between 0 and 1).

The function should return an SArray, where each element is set to +1 or -1 depending whether the corresponding probability exceeds `threshold`.


```python
def apply_threshold(probabilities, threshold):
    ### YOUR CODE GOES HERE
    # +1 if >= threshold and -1 otherwise.
    return probabilities.apply(lambda x: +1 if x >= threshold else -1)
```

Run prediction with `output_type='probability'` to get the list of probability values. Then use thresholds set at 0.5 (default) and 0.9 to make predictions from these probability values.


```python
probabilities = model.predict(test_data, output_type='probability')
predictions_with_default_threshold = apply_threshold(probabilities, 0.5)
predictions_with_high_threshold = apply_threshold(probabilities, 0.9)
```


```python
print("Number of positive predicted reviews (threshold = 0.5): %s" % (predictions_with_default_threshold == 1).sum())
```

    Number of positive predicted reviews (threshold = 0.5): 28897



```python
print("Number of positive predicted reviews (threshold = 0.9): %s" % (predictions_with_high_threshold == 1).sum())
```

    Number of positive predicted reviews (threshold = 0.9): 25031


**Quiz Question**: What happens to the number of positive predicted reviews as the threshold increased from 0.5 to 0.9? Decrease

## Exploring the associated precision and recall as the threshold varies

By changing the probability threshold, it is possible to influence precision and recall. We can explore this as follows:


```python
# Threshold = 0.5
precision_with_default_threshold = turicreate.evaluation.precision(test_data['sentiment'],
                                        predictions_with_default_threshold)

recall_with_default_threshold = turicreate.evaluation.recall(test_data['sentiment'],
                                        predictions_with_default_threshold)

# Threshold = 0.9
precision_with_high_threshold = turicreate.evaluation.precision(test_data['sentiment'],
                                        predictions_with_high_threshold)
recall_with_high_threshold = turicreate.evaluation.recall(test_data['sentiment'],
                                        predictions_with_high_threshold)
```


```python
print("Precision (threshold = 0.5): %s" % precision_with_default_threshold)
print("Recall (threshold = 0.5)   : %s" % recall_with_default_threshold)
```

    Precision (threshold = 0.5): 0.941239575042392
    Recall (threshold = 0.5)   : 0.9681082043068162



```python
print("Precision (threshold = 0.9): %s" % precision_with_high_threshold)
print("Recall (threshold = 0.9)   : %s" % recall_with_high_threshold)
```

    Precision (threshold = 0.9): 0.9728736366905038
    Recall (threshold = 0.9)   : 0.8667734472326036


**Quiz Question (variant 1)**: Does the **precision** increase with a higher threshold? Yes

**Quiz Question (variant 2)**: Does the **recall** increase with a higher threshold? No

## Precision-recall curve

Now, we will explore various different values of tresholds, compute the precision and recall scores, and then plot the precision-recall curve.


```python
threshold_values = np.linspace(0.5, 1, num=100)
print(threshold_values)
```

    [0.5        0.50505051 0.51010101 0.51515152 0.52020202 0.52525253
     0.53030303 0.53535354 0.54040404 0.54545455 0.55050505 0.55555556
     0.56060606 0.56565657 0.57070707 0.57575758 0.58080808 0.58585859
     0.59090909 0.5959596  0.6010101  0.60606061 0.61111111 0.61616162
     0.62121212 0.62626263 0.63131313 0.63636364 0.64141414 0.64646465
     0.65151515 0.65656566 0.66161616 0.66666667 0.67171717 0.67676768
     0.68181818 0.68686869 0.69191919 0.6969697  0.7020202  0.70707071
     0.71212121 0.71717172 0.72222222 0.72727273 0.73232323 0.73737374
     0.74242424 0.74747475 0.75252525 0.75757576 0.76262626 0.76767677
     0.77272727 0.77777778 0.78282828 0.78787879 0.79292929 0.7979798
     0.8030303  0.80808081 0.81313131 0.81818182 0.82323232 0.82828283
     0.83333333 0.83838384 0.84343434 0.84848485 0.85353535 0.85858586
     0.86363636 0.86868687 0.87373737 0.87878788 0.88383838 0.88888889
     0.89393939 0.8989899  0.9040404  0.90909091 0.91414141 0.91919192
     0.92424242 0.92929293 0.93434343 0.93939394 0.94444444 0.94949495
     0.95454545 0.95959596 0.96464646 0.96969697 0.97474747 0.97979798
     0.98484848 0.98989899 0.99494949 1.        ]


For each of the values of threshold, we compute the precision and recall scores.


```python
precision_all = []
recall_all = []

probabilities = model.predict(test_data, output_type='probability')
for threshold in threshold_values:
    predictions = apply_threshold(probabilities, threshold)
    
    precision = turicreate.evaluation.precision(test_data['sentiment'], predictions)
    recall = turicreate.evaluation.recall(test_data['sentiment'], predictions)
    
    precision_all.append(precision)
    recall_all.append(recall)
```

Now, let's plot the precision-recall curve to visualize the precision-recall tradeoff as we vary the threshold.


```python
import matplotlib.pyplot as plt
%matplotlib inline

def plot_pr_curve(precision, recall, title):
    plt.rcParams['figure.figsize'] = 7, 5
    plt.locator_params(axis = 'x', nbins = 5)
    plt.plot(precision, recall, 'b-', linewidth=4.0, color = '#B0017F')
    plt.title(title)
    plt.xlabel('Precision')
    plt.ylabel('Recall')
    plt.rcParams.update({'font.size': 16})
    
plot_pr_curve(precision_all, recall_all, 'Precision recall curve (all)')
```


![png](output_56_0.png)


**Quiz Question**: Among all the threshold values tried, what is the **smallest** threshold value that achieves a precision of 96.5% or better? Round your answer to 3 decimal places. 0.813


```python
i = 0
while precision_all[i] < 0.965:
    i += 1
threshold_values[i]
```




    0.8131313131313131



**Quiz Question**: Using `threshold` = 0.98, how many **false negatives** do we get on the **test_data**? (**Hint**: You may use the `turicreate.evaluation.confusion_matrix` function implemented in Turi Create.) 9056


```python
predictions = apply_threshold(probabilities, 0.98)
confusion_matrix = turicreate.evaluation.confusion_matrix(test_data['sentiment'], predictions)
confusion_matrix
```




<div style="max-height:1000px;max-width:1500px;overflow:auto;"><table frame="box" rules="cols">
    <tr>
        <th style="padding-left: 1em; padding-right: 1em; text-align: center">target_label</th>
        <th style="padding-left: 1em; padding-right: 1em; text-align: center">predicted_label</th>
        <th style="padding-left: 1em; padding-right: 1em; text-align: center">count</th>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">-1</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">9056</td>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">19039</td>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">-1</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">-1</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">4943</td>
    </tr>
    <tr>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">-1</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">1</td>
        <td style="padding-left: 1em; padding-right: 1em; text-align: center; vertical-align: top">298</td>
    </tr>
</table>
[4 rows x 3 columns]<br/>
</div>



This is the number of false negatives (i.e the number of reviews to look at when not needed) that we have to deal with using this classifier.

# Evaluating specific search terms

So far, we looked at the number of false positives for the **entire test set**. In this section, let's select reviews using a specific search term and optimize the precision on these reviews only. After all, a manufacturer would be interested in tuning the false positive rate just for their products (the reviews they want to read) rather than that of the entire set of products on Amazon.

## Precision-Recall on all baby related items

From the **test set**, select all the reviews for all products with the word 'baby' in them.


```python
baby_reviews =  test_data[test_data['name'].apply(lambda x: 'baby' in x.lower())]
```

Now, let's predict the probability of classifying these reviews as positive:


```python
probabilities = model.predict(baby_reviews, output_type='probability')
```

Let's plot the precision-recall curve for the **baby_reviews** dataset.

**First**, let's consider the following `threshold_values` ranging from 0.5 to 1:


```python
threshold_values = np.linspace(0.5, 1, num=100)
```

**Second**, as we did above, let's compute precision and recall for each value in `threshold_values` on the **baby_reviews** dataset.  Complete the code block below.


```python
precision_all = []
recall_all = []

for threshold in threshold_values:
    
    # Make predictions. Use the `apply_threshold` function 
    ## YOUR CODE HERE 
    predictions = apply_threshold(probabilities, threshold)

    # Calculate the precision.
    # YOUR CODE HERE
    precision = turicreate.evaluation.precision(baby_reviews['sentiment'], predictions)
    
    # YOUR CODE HERE
    recall = turicreate.evaluation.recall(baby_reviews['sentiment'], predictions)
    
    # Append the precision and recall scores.
    precision_all.append(precision)
    recall_all.append(recall)
```

**Quiz Question**: Among all the threshold values tried, what is the **smallest** threshold value that achieves a precision of 96.5% or better for the reviews of data in **baby_reviews**? Round your answer to 3 decimal places. 0.848


```python
i = 0
while precision_all[i] < 0.965:
    i += 1
round(threshold_values[i],3)
```




    0.848



**Quiz Question:** Is this threshold value smaller or larger than the threshold used for the entire dataset to achieve the same specified precision of 96.5%? larger

**Finally**, let's plot the precision recall curve.


```python
plot_pr_curve(precision_all, recall_all, "Precision-Recall (Baby)")
```


![png](output_74_0.png)

