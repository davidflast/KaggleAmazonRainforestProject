# Kaggle Amazon Rainforest Deforestation Detection 
Deforestation in the Amazon causes climate change and biodiversity loss and is caused by both human activity and natural events. This kaggle competition provided a dataset of satellite imagery labeled with weather conditions, land cover, and deforestation events to better understand environmental degradation. The dataset contains 40,000 training examples and is very imbalanced with significantly more data for basic land features like primary forests and much fewer for environmental violations like logging, mines etc. Recall is more important for this task since we are trying to catch a smaller number of violations in a large forest and can handle false positives with human review. Therefore, I use F2 score across the whole dataset to evaluate the model, while also tracking the recall on the less common classes. 

The baseline model is a ResNet50 pretrained on ImageNet. The pretrained layers are allowed to be optimized since satellite imagery is very different than the images in ImageNet. Images are RGB jpgs that are transformed to 224X224 pixels and normalized to the ImageNet dataset. I flatten the outputs from the ResNet50 convolutional layers to one dimension using adaptive average pooling. The fully connected layers are replaced with a single fully connected hidden layer with ReLU and dropout of 0.25, and an output layer with a single output for each class. The scores pass through a sigmoid activation layer with a threshold of 0.5 to classify the output. The model was train with binary cross entropy loss with an adam optimizer with L2 regularization. The training runs for a maximum of 10 epochs with a 1e-4 learning rate and early stopping after 3 epochs of no improvement on F2 score. The best model was achieved after 2 epochs with an F2 of 0.882. The model achieved good performance on the common classes, but had very low recall on the rare classes. 

I experimented with a few enhancements to this model. First, I tested using focal loss to deal with class imbalance, which is a variant of binary cross entropy loss that down weights the loss from easily classified examples. This improved F2 to 0.886, but the model trained for more epochs since the more common examples trained slower than with BCE. The recall did not improve on rare classes, so I did not choose to use this enhancement.  Next, I tested augmenting the data with random vertical/horizontal flips and rotations, which improved F2 1% to 0.89. To handle class imbalance, I tested using a sampler that picks a class at random, then chooses an example from that class. This reduced F2 score to 0.88, but it improved the recall on the rare classes making it a worthwhile change. I then theorised that the model was learning easy examples quickly, but needed more time to learn the more difficult examples. I trained the model for 20 epochs, achieving the best score at epoch 12 of 0.882. This was not enough of an enhancement to be worth it and was potentially overfitting. Finally, I tested optimizing the threshold for the class sample + augmentation approach and achieved an F2 of 0.902 by lowering the threshold to 0.19.

| Metric  | Baseline   | Focal Loss | Rotations  | Random Sampler + Rotations | Run 20 Epochs | Threshold .19 |
| --------| -------    | --------   | -------    | --------                   | -------       | -------       | 
| F2      | 0.882      | 0.886      |  0.89      |  0.88                      | 0.882         | 0.902         |

Long term, I would first dive deeper into data quality. Experts could check a stratified random sample of images, evaluate error rate, and see if there are any systemic issues with labeling. Next, I would use [cleanlab](https://github.com/cleanlab/cleanlab/) to identify likely misclassified examples to clean the dataset. Next I would do a deeper analysis of the dataset and current models. I would chart the correlations between labels, check if the model performance degrades under hazy or cloudy conditions, and chart a confusion matrix for each class. To improve the modeling, I would use my current modeling strategy and run a grid hyperparameter search on batch size, learning rate, network depth (ie. resnet 18, 50, 101, 152). If I had more limited time or resources, I would use bayesian hyperparameter optmization instead. I would also test a vision transformer model to see if the extra size improves model performance. One weakness of the current approach is I am using pretraining from ImageNet which may not transfer well to satellite imagery. I would gather a large sample (1 million+) of unlabeled satellite images of the same resolution and then use a [vision transformer masked auto encoding](https://huggingface.co/docs/transformers/main/model_doc/vit_mae) approach to train the embeddings. The dataset contains significantly more clear images than hazy or partly cloud ones. I would try a data augmentation to make clear images hazy in order to expand those classes. This competition uses an image classification scheme to catch environmental degradation, but following the [results from RegLab](https://arxiv.org/abs/2208.08919), object detection or segmentation models may perform better at this task. Finally, this dataset uses imagery with only RGB bands from one image, but using multiple bands taken at multiple times with a pretrained model like [Prithvi from IBM and Nasa](https://huggingface.co/ibm-nasa-geospatial) could improve performance. 

If I was deploying this to the real world. First, I would reduce costs by deploying a smaller model on TPU instances. Next, we should test the model's predictions with site visits to evaluate if the predictions are accurate. If those site visits are expensive (which they likely are in the Amazon), I would switch to selecting the model and thresholds by f0.5 score instead of f2 to ensure higher likelihood of catching degradation. Finally, I would setup monitors to check for data distribution shifts, and also manualy sample predictions on a yearly or monthly cadence to ensure the model performance does not degrade. This would catch if classes shift due to environmental changes or changes in human activity and allows us to retrain the model to adapt to those changes. 

 In this weekend project, I tested a baseline off the shelf ResNet50 model which performed well overall, but failed to catch rare deforestation events. I tested enhancements to sample rare classes in a balanced manner, augment the dataset with rotations, and optimize the threshold for the preferred metric of recall, all of which improved the recall on deforestation events substatially. These methods were able to catch some events of deforestation, but more work is required to improve the model. This project demonstrates that satellite imagery with computer vision can be used to detect deforestation in the Amazon Rainforest. 


### Kaggle Limitations
* Tiff images no longer availible for this dataset so I used jpgs. 
* Logging for my experiments was saved to MLFlow, but kaggle deletes the workspace every time I restart the notebook so its no longer saved. 
* I could not find the labels for the test set, so could not do an analysis of the model accuracy on the test set. 