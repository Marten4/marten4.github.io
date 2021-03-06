---
layout: post
title:  Pseudo features
categories: [Machine Learning]
mathjax: true
published: false
---
*This is an old post that I kept unpublished till now.*

Every classification problem has some "true" set of features. These features, if given, renders the problem almost entirely solved. Unfortunately,  there is no way to directly uncover these true features. Instead we use a labeled training set to learn a mapping function from the inputs to the labels. The hope is that by learning the mapping, we would also learn at least some or most of these true features. However, the problem is that, the training set may also contain features that are not part of the true set but do help with the mapping. This most often happens because of artifacts in data collection. I'll refer to these type of features as pseudo features.

<!-- more -->

**Definition:** Pseudo features are features that have a high predictive power at training time but either do not provide any help or hurt with generalization.

To give an intuitive example, let's say that we're working on image classification and for some reason one of the classes has a bias in the training data. For example all of the pictures that have a bear on it were taken in a snowy background. Since the model is trying to find features that are correlated with the label it *may* learn that a lot of white pixels means that there's a bear on the picture. Obviously, this is an oversimplification of the problem but hopefully you understand the point I'm trying to make.
<br>
<center>
<tr>
<td> <img src="/images/snow_bear.jpg" alt="Bear" style="width: 250px" border="1"/> </td>
<td> <img src="/images/snow_no_bear.jpg" alt="No Bear" style="width: 250px" border="1"/> </td>
</tr>
</center>
<br>

Unfortunately, these kinds of data biases are not rare and they are extremely difficult to catch. Rolling out models in the real-world with the intention of using them for critical decisions is scary if you don't know for sure that the model is not using pseudo features for predictions.

One potential solution to this problem could be to expand the objective of training. For classification, we usually provide labeled training data $$(x, y) \in \mathcal{D}$$ and optimize the parameters $$\theta$$ of the model $$f$$ such that the outputs match the labels. This objective promotes features that help with prediction on the training set. But as we've discussed earlier, pseudo features can also be promoted during this process. Instead of only promoting features with the training set, we could also try to demote features using some auxiliary dataset $$\hat{\mathcal{D}}$$. The auxiliary dataset should ideally be a dataset that contains the pseudo features but does not contain any true features. This would mean that a model that is making predictions with the true features should be *confused* on the inputs that are from $$\hat{\mathcal{D}}$$. The two term objective is:

$$\min_{\theta} \mathbb{E}_{x \sim \mathcal{D}} \text{L}(y, f(x; \theta)) + \lambda\mathbb{E}_{x \sim \hat{\mathcal{D}}}\text{H}(f(x; \theta))$$

Where $$\text{L}$$ is some classification loss, for example cross-entropy loss and $$\text{H}$$ is entropy.


<!-- Let's call these instances $$D_{in}$$, for every instance $$x_i \in D_{in}$$, we have a corresponding class label $$y_i \in Y$$. But we also have access to $$D_{out}$$, a set that contains items that are not part of $$D_{in}$$. For example if our task is 3 way document classification with the categories sports, politics and finance. The everything else could be documents about entertainment, fashion, high tech etc. Or we could expand the definition even more and include everything in the universe that is not $$D_{in}$$. We could use the instances in $$D_{out}$$ to push the model to be uncertain on those examples. Which intuitively makes sense, if a model is trained to recognize hand-written digits and we show it a picture of the letter "A", the desired output of the classifier should be maximum uncertainty over the output choices (digits from 0 to 9). -->

<!-- This objective has come up in recent years in the context of OOD detection [[1]](#1)[[2]](#2). They use $$D_{out}$$ to supervise the model on detecting OOD samples. And they have shown this method to be successful at that. But I think there is potential for another outcome. Going back to our example with the bear, What if we expose the model to images that have a snowy background but do not have an object of our class in the image. We then push the model to be uncertain on those images. The model would (hopefully) learn that white pixels $$\neq$$ bear and that it needs to 'pick up' other features to be able to classify the pictures with a bear on it. In other words, OOD data can potentially lead to a more robust solution by discarding the pseudo features. But we need to slightly change what we call $$D_{out}$$, it need not be OOD. Furthermore, not every OOD instance belongs to $$D_{out}$$. A more accurate description of the samples that we seek is out-of-concept (OOC) instances. The same concept (images of bears) can have different distributions depending on how the data was collected. We care about the concept of interest not being present in $$D_{out}$$. -->

<!-- Caveat: robustness is hard to measure if the test set also contains the pseudo features. -->

To check if this idea has any merit I designed a super simple experiment. The idea was to create the most ideal scenario. If this method doesn't work for this, it wouldn't work anything.

To start, I define a boolean function:

$$(x_1 \land \neg x_2) \land (x_3 \lor x_4) \lor x_1 \lor (x_5 \land x_6) \land \neg x_7$$

The reason I'm using this function is because it has a big enough sample space $$(2^7)$$ to make it possible to learn the function from a random sub-sample and the output space is roughly balanced (about half are zeros and half are ones). The function is also linear, so we can learn it with a linear function. To construct a dataset we create a truth table and label all of the outputs with the defined function. To verify that the dataset is learnable, I fit a linear classifier on 50% of the dataset and test it on the rest. The results show that the function is indeed recoverable from only half the data points. I ran 10 trials with different splits each time and get about ~0.945 accuracy on the test set with a standard deviation of ~0.03. This is the ceiling on the performance.

For the second part of the experiment I add a pseudo feature to the dataset. This feature is perfectly correlated with the label in the training set but is randomized in the test set. The train and test set are created by equally splitting the original dataset. I again fit a linear classifier on this modified dataset. I run 100 trials and report the mean accuracy and the standard deviation on both the train and the test set. As expected the results show that there is a perfect, 1.00 accuracy on the training set but the performance on the test set drops all the way to ~0.64 with a standard deviation of ~0.10. This is explained by the fact that the pseudo feature has a perfect correlation with the label in the training set but is randomized in the test set. At training time the model ends up relying on this feature too much, which causes it to suffer at test time. We can verify this by observing the magnitude of the coefficient corresponding to that feature.

Finally, I create a OOD dataset that is entirely zeros expect for the feature that corresponds to the pseudo feature. The modified loss function will contain a term that will prefer a low KL divergence on the output of the model on these examples and the uniform distribution. In other words, the objective will force the model to be uncertain on OOD samples.

The results after 100 random trials show that the new objective helps regularize the model by essentially canceling the pseudo feature. The test set accuracy is ~0.91 with a standard deviation of ~0.03. The accuracy isn't as high as the model that does not get the pseudo feature at all, but it is definitely much higher than 0.64. Furthermore, the weight of the pseudo feature is consistently low, compared to the baseline model.

## References
<a id="1">[1]</a>
Lee et al.
[Training Confidence-calibrated Classifiers for Detecting Out-of-Distribution Samples.](https://arxiv.org/pdf/1711.09325.pdf)
ICLR 2018.

<a id="2">[2]</a>
Hendrycks et al.
[Deep Anomaly Detection with Outlier Exposure.](https://arxiv.org/pdf/1812.04606.pdf)
ICLR 2019.
