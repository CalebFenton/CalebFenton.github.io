title: "Measuring the Usefulness of Multiple Models"
tags:
  - machine learning
  - research
comments: true
date: 2017-09-28 00:13:37
---

The past several years have seen a massive increase in products, services, and features which are powered or enhanced by artificial intelligence — voice recognition, facial recognition, targeted advertisements, and so on. In the anti-virus industry, we’ve seen a similar trend with a push away from traditional, signature-based detection towards fancy machine learning models. Machine learning allows anti-virus companies to leverage large amounts of data and clever feature engineering to build models which can accurately detect malware and scales much better than manual reverse engineering. Using machine learning and training a model is [simple enough](https://github.com/llSourcell/antivirus_demo) that there are YouTube videos like “[Build an Antivirus in 5 Minutes](https://www.youtube.com/watch?v=iLNHVwSu9EA&feature=youtu.be)“. While these home brew approaches are interesting and educational, building and maintaining an effective, competitive model takes a _lot_ longer than 5 minutes.
<!-- more -->

To stay competitive with machine learning one must constantly beef up the training data to include new and diverse samples, engineer better features, refine old features, tune model [hyper parameters](https://images.gr-assets.com/hostedimages/1467729070ra/19623361.gif), research new models, and so on. To this end, I’ve been researching how to use multiple models to improve detection accuracy. The intuition is that different models may have different strengths and weaknesses and as the number of models which agree increases, you can be more certain. Using multiple model’s isn’t new. It’s a common technique generally agreed to improve accuracy, and we already use it in some places. However, I wanted to quantify how much can it improve accuracy and which combination of models would work best. In this post, I’m going to share my findings and give some ideas for future research.

## Selecting the Algorithms

If you want to use multiple models, the first question is which learning algorithms to use. Intuitively, you want strong models which make different mistakes. For example, if one model is wrong about a particular file, you want your other models to not be wrong. In other words, you want to minimize the size of the intersection of the sets of misclassified files between models. In this way, three strong models which make the same mistakes may not perform as well as one strong model with two weaker models which make entirely different mistakes.

I picked three models which I hoped would perform differently: [random forest](http://scikit-learn.org/stable/modules/generated/sklearn.ensemble.RandomForestClassifier.html#sklearn.ensemble.RandomForestClassifier), [multi-layer perceptron](http://scikit-learn.org/stable/modules/generated/sklearn.neural_network.MLPClassifier.html), and [extra trees](http://scikit-learn.org/stable/modules/generated/sklearn.ensemble.ExtraTreesClassifier.html). Random forests, [which I’ve explained previously](https://sentinelone.com/2016/12/05/detecting-malware-pre-execution-static-analysis-machine-learning/) and extra trees are both ensemble algorithms which use decision trees as a base estimator, but I figured I could use very different parameters for each and get different results.

For evaluating the performance of multiple models, consider that when a model judges a file, there are four outcomes:

1.  true positive (TP) – file is bad and model says bad
2.  true negative (TN) – file is good and model says good
3.  false positive (FP) – file is good but model says bad
4.  false negative (FN) – file is bad but model says good

In the anti-virus industry, the most important metric for a model is the FP rate. If you detect 100% of malware but have an FP rate of only 0.1%, you’ll still be deleting 1 in 1,000 benign files which will _probably_ [break something important](https://1.bp.blogspot.com/--w1K7n5WMLM/UoKA3t9CugI/AAAAAAAD_jw/awBs5Xd6PI4/s1600/a_few_things_that_are_just_seconds_from_disaster_35.gif) (like the operating system). The second most important metric is the TP rate, or the detection rate. This is how many malicious files you detect and these two rates are usually antagonistic. Improving the TP rate usually means increasing the FP rate, and vice versa.

Since FPs are so important to avoid, I decided to evaluate model combinations by measuring how much the FP sets overlap. The less they overlap, the better. This isn’t very rigorous, but it’s _fast_. I prefer to get quick results, build my intuition, and get more rigorous as I iterate. In a perfect world with unlimited time, CPU power, and memory, I would setup a grid search to find the ideal parameters for a dozen models and then build another grid search to find the ideal way to combine the models. Unfortunately, this could take weeks. By doing some quick experiments, I can find out if the idea is worth perusing, possibly come up with a better test, and save a lot of time by eliminating certain possibilities.

## Building the Models

The training data consisted of features from a wide variety of about 1.7 million executables, about half of which were malicious. The data were vectorized and prepared by removing invariant features, normalizing, scaling, and agglomerating features. Decision trees don’t care much about scaling and normalizing, but MLP and other models do. By limiting the number of features to 1000, training time is reduced and previous experiments have shown that it doesn’t degrade model performance much. Below is the code for preparing the matrix:

```python
import sklearn as skl
import sklearn.feature_selection
import gc

variance = skl.feature_selection.VarianceThreshold(threshold=0.001)
matrix = variance.fit_transform(matrix)

normalize = sklearn.preprocessing.Normalizer(copy=False)
matrix = normalize.fit_transform(matrix)

# Converts matrix from sparse to dense
scale = skl.preprocessing.RobustScaler(copy=False)
matrix = scale.fit_transform(matrix.toarray())

# Lots of garbage to collect after last step
# This may prevent some out of memory errors
gc.collect()

fa = sklearn.cluster.FeatureAgglomeration(n_clusters=1000)
matrix = fa.fit_transform(matrix)
```

The random forest (RF), extra trees (ET), and multi-layer perceptron (MLP) models were built using the [SKLearn](https://github.com/scikit-learn/scikit-learn) Python library from the prepared matrix.

**NOTE:** I've since learned it's much better to use [MaxAbsScaler](http://scikit-learn.org/stable/modules/generated/sklearn.preprocessing.MaxAbsScaler.html) with sparse feature matricies. Read this for an explanation why: [Right function for normalizing input of sklearn SVM](https://stackoverflow.com/questions/30918781/right-function-for-normalizing-input-of-sklearn-svm). Look here for some example code: [normalization.py](https://gist.github.com/CalebFenton/66aa04af7b4a4d98efca059cb8c2e7aa#file-normalization-py)

## Testing the Models

The strongest performing model was the random forest with extra trees coming in a close second. The lackluster MLP performance is likely due to bad tuning of hyper parameters but it could just be a bad algorithm for this type of problem.

Below is the table of results showing the number of FPs from each model and the number of FPs in the intersection between each model and every other model:

|Model|FPs|∩ RF|∩ MLP|∩ ET|
|-----|---|----|-----|----|
|RF|3928|-|3539|2784|
|MLP|104356|3539|-|3769|
|ET|4302|2784|3769|-|

The FPs common between all models is 2558\. This means that all three models mistakenly labeled 2558 of the files as malicious when they were [actually](http://ct.fra.bz/ol/fz/sw/i59/2/8/3/frabz-ACTUALLY-9f1b65.jpg) benign. The best way to minimize FPs is to require that all three models agree a file is malicious. With these three models, this would decrease the false positive rate by 35%. For example, instead of using just a random forest and getting 3928 FPs, if all models had to agree, the FPs would be limited to just 2558.

Requiring all models agree is a highly conservative way to combine models and is the most likely to reduce the TP rate. To measure the TP rate reduction, I checked the size of the intersections of TPs between the models. The table below shows the figures:

|Model|TPs|∩ RF|∩ MLP|∩ ET|
|-----|---|----|-----|----|
|RF| 769043 |-| 759321 | 767333 |
|MLP| 761807 | 759321 |-| 758488 |
|ET| 768090 | 767333 | 758488 |-|

As with FPs, the RF model performed the best with the ET model lagging slightly behind. The intersection of TPs between all models was 757880\. If all models were required to agree a file was malicious, this means the TP rate would only decrease 1.5%.

Below is roughly the code I used to collect FPs and TPs:

```python
# matrix contains vectorized data
# indicies contains array of sample sha256 hashes
# labels contains array of sample labels - True=malicious, False=benign
def get_tps(labels, predicted, indicies):
    tps = set()
    for idx, label in enumerate(labels):
        prediction = predicted[idx]
        if label and prediction:
            tps.add(indicies[idx])
    return tps

def get_fps(labels, predicted, indicies):
    fps = set()
    for idx, label in enumerate(labels):
        prediction = predicted[idx]
        if not label and prediction:
            fps.add(indicies[idx])
    return fps

# Fit the classifier
mlp = skl.neural_network.MLPClassifier()
mlp.fit(matrix, labels)

# Make predictions and collect FPs and TPs
mlp_predicted = skl.model_selection.cross_val_predict(mlp, matrix, labels, cv=10)
mlp_fps = get_fps(labels, predicted, indicies)
mlp_tps = get_fps(labels, predicted, indicies)

# rf_fps contains random forest FPs
print(rf_fps.intersection(mlp_fps))
```

## Conclusion

This research suggests that by using multiple models one could easily reduce FPs by about a third with only a tiny hit to the detection rate. This seems promising, but the real world is often more complicated in practice. For example, we use many robust, non-AI systems to avoid FPs so the 35% reduction likely wouldn’t affect all file types equally and most of the FP reduction might be covered by such pre-existing systems. This research only establishes an upper bound for FP reductions. There’s also engineering considerations for model complexity and speed that need to be taken into account. I’m using Python and don’t care how slow models are, but implementing the code in C and making it performant might be quite tricky.

The extra trees worked better than expected. I did very little tuning yet it was strong and had fairly different false positives than the random forest model. The MLP model may, with enough tuning, eventually perform well, but maybe it’s a bad model for the number and type of features. I’m eager to try with an SVC model, but SVC training time grows quadratically so I’d need to use a [bagging classifier with many smaller SVCs](https://stackoverflow.com/questions/31681373/making-svm-run-faster-in-python).

There are many different ways to combine models — voting, soft voting, blending, and stacking. What I’ve described here is a variant of voting. I’d like to experiment with stacking which works by training multiple models, then using the output of those models as features into a “second layer” model which figures out how to best combine the models. Since I’m most interested in minimizing false positives, I’ll have to compare stacking performance versus requiring all models agree. It may be possible to weight benign samples so models favor avoiding false positives while training without sacrificing detection rates.

The main bottleneck for this research is computing speed and memory. I may be able to just use a smaller training set. I can find out how small the training set can be by training on a subset of the data and testing the performance against the out-of-sample data. Another option is to switch from SKLearn to TensorFlow GPU which allows me to take advantage of my [totally justified](http://www.cnn.com/2011/TECH/gaming.gadgets/01/31/video.games.smarter.steinberg/index.html) video card purchase.