# Temporal subgraph classification
Network classification has a significant role in detecting communities and recognizing patterns within the networks.  
Most existing work in this area heavily focuses on embedding the networks, without providing much information on the best way of sampling it into subgraphs in the first place.  

Here, I focus on two alternate views of sampling a temporal network into subgraphs in order to achieve maximum classification accuracy.  
Experimental results show model improvement varying between 10 and 20%, depending on the network’s density.

## 1. Problem formulation 
I use definitions of temporal networks and temporal network motifs as defined in [Paranjape, A., Benson, A.R., Leskovec, J.: Motifs in Temporal Networks. arXiv:1612.09259]. Due to space constraints, here are examples on static and temporal motifs in the following figure, and refer interested readers to the paper for further detail.
![Motif comparison](https://github.com/ZafirStojanovski/temporal-subgraph-classification/blob/master/motif%20comparison.jpg "Motif comparison")
Denote ![formula](https://github.com/ZafirStojanovski/temporal-subgraph-classification/blob/master/formula.png "formula") as a set of *N* subgraphs, where *V* is a set of nodes and *E* is a set of timestamped edges in *G*. Suppose that subgraphs can be categorized in *D* classes, where *D* < *N*. We associate *G* with a label of *L* ∈ {1, ..., *D*}.  
The idea of network classification is to assign novel subgraphs 𝐺𝑖 to one of the *𝐷* classes (subnetworks).

I will work with two datasets:  
1. E-mail network (Directed temporal graph, nodes are people and edges are emails sent from employee A to employee B at timestamp t)
2. Stack Exchange network (Directed temporal graph, nodes are users and edges are replies/comments from user A to user B at timestamp t)  

Both of them are constructed from 4 subnetworks (classes) each.
![Datasets] (https://github.com/ZafirStojanovski/temporal-subgraph-classification/blob/master/datasets.jpg "Datasets")
## 2. Sampling
As for any machine learning problem, we first need to define what the samples are. In this case, we only have the network as a whole for each of the classes. So, we need to break it up.  
A fairly reasonable way to partition the network is to break it into time-windows with variable sizes that span fixed amount of time - delta.  

Simply put, I first sort the network (which is a list of timestamped edges) by the timestamp value, then I form time-windows that span 24-hour period (our choice of delta).
From here, I build 2 datasets:  
* each sample is a single window
* each sample is a concatenation of itself and its two neighboring windows

![Alternate ways of sampling](https://github.com/ZafirStojanovski/temporal-subgraph-classification/blob/master/btr.png "Alternate ways of sampling")  

The first alternative can hurt model performance in a sense that it loses sense of context temporality as a result of the discretization performed on the graph - any information in the neighboring time-frames is lost.
However, the second alternative will produce samples with more context - what was happening in the previous time-frame and what is going on in the future time-frame.  
Not only that, but we preserve the overall number of samples in the dataset - if we were to only concatenate each triplet of time-windows without overlapping, we would end up with samples that have as much temporal context as the second alternative, but the resulting dataset would have 3 times less samples than the one produced in the second alternative, which will definitely affect the models since we're already dealing with limited number of samples.  

## 3. Embedding
Now that we have defined what our samples are, we need to embed them - transform them into vectors.  
I decided to sharpen Ockham’s razor and test the three simplest embedding techniques:  
1. **Temporal motif distribution** - a vector of 36 values, each one representing the count of the temporal motifs described in the [Paranjape, A., Benson, A.R., Leskovec, J.: Motifs in Temporal Networks. arXiv:1612.09259]
2. **Static motif distribution** - a vector of 13 values, each one representing the count of the static motifs described in [Kun Tu, Jian Li, Don Towsley, Dave Braines, Liam D. Turner: Network Classification in Temporal Networks Using Motifs. arXiv:1807.03733]
3. A **combination** of the previous two  

## 4. Modeling
Once we transform each sample (subgraph) into a feature vector, we are ready to train the machine learning models.  
I perform 80-20 train/test split, in such manner that the models are trained on past time-frames and test on future ones.
We train three different models which are known to do well on classification tasks with relatively small datasets: Support vector machines, Random forests and XG Boost.  
Using 10-fold CV Grid Search we look for the best hyperparameters for each model and then using the best estimator we evaluate the test set.  

![Results](https://github.com/ZafirStojanovski/temporal-subgraph-classification/blob/master/results.png "Results") 

## Conclusion
From the plots, we can confirm our hypothesis: for "sparse" temporal networks (ones that have fewer interactions between the nodes in a fixed time window) like the E-mail one, the method of windowing improved our model performance up to 20%.  
This is because the process of motif construction in the network spans more than the defined time-frame so the windowing technique managed to capture more context from its neighbors.  
On the other hand, for "dense" temporal networks like the Stack Exchange one, the windowing technique had a modest improvement of 5-8%. This is due to the fact that its motifs mostly occur in the specified time-window and only small amount of information can be extracted from the neighboring windows.  
Meaningful e-mail communication usually doesn't occur in just one day (delta value of our time frame), so that is why windowing the timeframes improved our accuracy so much - we were able to capture much more context of the communication between the employees in the department, thus increasing the number of motif formations.  
On the other hand, Stack Exchange discussions are rarely revisited after the day they were originally posted, so that is why we achieved such modest improvements by windowing the timeframes.  
