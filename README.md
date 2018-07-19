# FAR-HO

Gradient-based hyperparameter optimization and meta-learning package based on [TensorFlow](https://www.tensorflow.org/)

This is the new package that implements the algorithms presented in the paper
 [_Forward and Reverse Gradient-Based Hyperparameter Optimization_](http://proceedings.mlr.press/v70/franceschi17a). For the older package see [RFHO](https://github.com/lucfra/RFHO). FAR-HO [features simplified interfaces, additional
capabilities and a tighter integration with `tensorflow`](https://github.com/lucfra/FAR-HO#new-features-and-differences-from-rfho). 

- Reverse hypergradient (`ReverseHG`), generalization of algorithms presented in Domke [2012] and MacLaurin et Al. [2015] (without reversable dynamics and "reversable dtype")
- Forward hypergradient (`ForwardHG`)
- Online versions of the two previous algorithms: Real-Time HO (RTHO) and Truncated-Reverse HO (TRHO)
- Implicit differentiation (`ImplicitHG`), can be used to implement HOAG algorithm [Pedregosa, 2016] by setting the 
tolerance for solving the inner problem and computing the hypergradient as decreasing sequence 
(default value is exponentially decreasing)

The algorithms algorithms compute, with different procedures, the (approximate) gradient
  of an outer objective such as a validation error with respect 
  to the outer variables (e.g. hyperparameters). 
  We call the gradient of the outer objective _hypergradient_.
  The "online" algorithms perform "real time" (i.e. at training time) 
  outer variables updates, and are in general much faster then the "batch" versions,
  incurring, however, in a certain bias.
  
![alt text](https://github.com/lucfra/RFHO/blob/master/rfho/examples/0_95_crop.png 
"Response surface of a small neural network and optimization trajectory in the hyperparameter space.
The arrows depicts the negative hypergradient at the current point, computed with Forward-HG algorithm.")

These algorithms are useful also in meta-learning where parameters of various _meta-learners_ effectively play the role 
of  outer variables, as explained here in the workshop paper 
[_A Bridge Between Hyperparameter Optimization and Learning-to-learn_](https://arxiv.org/abs/1712.06283).
and [_Bilevel Programming for Hyperparameter Optimization and Meta-Learning_](http://proceedings.mlr.press/v80/franceschi18a/franceschi18a.pdf)

## Installation & Dependencies

Clone the repository and run setup script.

```
git clone git clone https://github.com/lucfra/FAR-HO.git
cd FAR-HO
python setup.py install
```

Beside "usual" packages (`numpy`), FAR-HO is built upon `tensorflow`. 
Some examples depend on the package [`experimet_manager`](https://github.com/lucfra/ExperimentManager)
while automatic dataset download (Omniglot) requires `datapackage`.

Please note that required packages will not be installed automatically.

## Overview

Aim of this package is to implement and develop gradient-based hyperparameter optimization (HO) techniques in
TensorFlow, thus making them readily applicable to deep learning systems. 
This optimization techniques find also natural applications in the field of meta-learning and
learning-to-learn. 
Feel free to issues comments, suggestions and feedbacks! You can email me at luca.franceschi@iit.it .


#### Quick Start 

- [Self contained example](https://github.com/lucfra/FAR-HO/blob/master/far_ho/examples/Example_weighted_error(and_lr_and_w0).ipynb) on MNIST with `ReverseHG` for the optimization of initial starting point (inital weights), weights of each example and learning rate. 
- _Coming soon_: expamples of application of online HO algorithms.
- _Coming soon_: What you can and cannot do with this package.
- [Hyper-representation](https://github.com/lucfra/FAR-HO/blob/master/far_ho/examples/hyper_representation.py) and related [notebook](https://github.com/lucfra/FAR-HO/blob/master/far_ho/examples/Hyper%20Representation_experiments.ipynb): an example in the context of learning-to-learn. In this case the hyperparameters are some of the weights of a convolutional neural network (plus the learning rate!). 
The idea is to learn a cross-episode shared representation by explicitly minimizing the mean generalization error over meta-training tasks. See [A bridge between hyperparameter optimization and learning-to-Learn](https://arxiv.org/abs/1712.06283) presentied at [Workshop on meta-learning](http://metalearning.ml/). _Note_: for the moment, for running the code for this experiment you need to install the package https://github.com/lucfra/ExperimentManager for data management and statistics recording. 
- See also [this experiments package](https://github.com/prolearner/hyper-representation) for code for reproducing few-shot experiments 
presented in ICML 2018 paper.

#### Core Steps

- Create a model<sup>1</sup> with TensorFlow
- Create the hyperparameters you wish to optimize<sup>2</sup> with the function `get_hyperparameter` (which could be also variables of your model)
- Define an inner objective (e.g. a training error) and an outer objective (e.g. a validation error) as scalar `tensorflow.Tensor`
- Create an instance of `HyperOptimizer` after choosing an hyper-gradient computation algorithm among
`ForwardHG`, `ReverseHG` and `ImplicitHG` (see next section)
- Call the function `HyperOptimizer.minimize` specifying passing the outer and inner objectives, 
as well as an optimizer for the outer problem (which can be any optimizer form `tensorflow`) 
and an optimizer for the inner problem (which must be an optimizer contained in this package; 
at the moment gradient descent, gradient descent with momentum and Adam algorithms are available, 
but it should be quite straightforward to implement other optimizers, email me if you're interested!) 
- Execute `HyperOptimizer.run(T, ...)` function inside a `tensorflow.Session`, 
optimize inner variables (parameters) and perform a step of optimization of outer variables (hyperparameter).

The two scripts the folder [autoMLDemos](https://github.com/lucfra/FAR-HO/tree/master/far_ho/examples/autoMLDemos) 
showcase typical usage of this package


```python
import far_ho as far
import tensorflow as tf

model = create_model(...)  

lambda1 = far.get_hyperparameter('lambda1', ...)
lambda1 = far.get_hyperparameter('lambda2', ...)
io, oo = create_objective(...)

inner_problem_optimizer = far.GradientDescentOptimizer(lr=far.get_hyperparameter('lr', 0.1))
outer_problem_optimizer = tf.train.AdamOptimizer()

farho = far.HyperOptimizer() 
ho_step = farho.minimize(oo, outer_problem_optimizer,
                     io, inner_problem_optimizer)

T = 100
with tf.Session().as_default():
  for _ in range(100):
    ho_step(T)    
```
____
<sup>1</sup> This is gradient-based optimization and for the computation
of the hyper-gradients second order derivatives of the training error show up
(_even tough no Hessian matrix is explicitly computed at any time_);
therefore, all the ops used
in the model should have a second order derivative registered in `tensorflow`.

<sup>2</sup> For the hyper-gradients to make sense, hyperparameters should be 
real-valued. Moreover, while `ReverseHG` should handle generic r-rank tensor 
hyperparameters, `ForwardHG`requires scalars hyperparameters. Use the keyword argument `scalar=True` in `get_hyperparameter` for obtaining a scalr splitting of a general tensor.

#### Which Algorithm Do I Choose?

Forward and Reverse-HG compute the same hypergradient, so
the choice is a matter of time versus memory!

![alt text](https://github.com/lucfra/RFHO/blob/master/rfho/examples/time_memory.png "Time vs memory requirements")

The online versions of the algorithms can dramatically speed-up the optimization.

#### The Idea Behind: Hyperparameter Optimization

The objective is to minimize some validation function _E_ with respect to
 a vector of hyperparameters _lambda_. The validation error depends on the model output and thus
 on the model parameters _w_. 
  _w_ should be a minimizer of the training error and the hyperparameter optimization 
  problem can be naturally formulated as a __bilevel optimization__ problem.  
   Since these problems are rather hard to tackle, we  
explicitly take into account the learning dynamics used to obtain the model  
parameters (e.g. you can think about stochastic gradient descent with momentum),
and we formulate
HO as a __constrained optimization__ problem. See the [paper](http://proceedings.mlr.press/v70/franceschi17a) for details.

#### New features and differences from RFHO

- __Simplified interface__: optimize paramters and hyperparamters with "just" a call of `far.HyperOptimizer.minimize`, create variables designed as hyperparameters with `far.get_hyperparameter`, no more need to vectorize the model weights, `far.optimizers` only need to specify the update as a list of pairs (v, v_{k+1})
- __Additional capabilities__: set an initalizaiton dynamics and optimize the (dsitribution) of initial weights, allowed explicit dependence of the outer objective w.r.t. hyperparameters, support for multiple outer objectives and multiple inner problems (episode batching, average the sampling from distributions, ...)
- __Tighter integration__: collections for hyperparameters and hypergradients (use `far.GraphKeys`), use out-of-the-box models (no need to vectorize the model), use any TensorFlow optimizer for the outer objective (validation error)
- Lighter package: only code for implementing the algorithms and running the examples
- Forward hypergradient methods have been reimplemented with a [double reverse mode trick](https://j-towns.github.io/2017/06/12/A-new-trick.html), thanks to Jamie Townsend. 

### Citing 


```latex
@InProceedings{pmlr-v70-franceschi17a,
  title = 	 {Forward and Reverse Gradient-Based Hyperparameter Optimization},
  author = 	 {Luca Franceschi and Michele Donini and Paolo Frasconi and Massimiliano Pontil},
  booktitle = 	 {Proceedings of the 34th International Conference on Machine Learning},
  pages = 	 {1165--1173},
  year = 	 {2017},
  volume = 	 {70},
  series = 	 {Proceedings of Machine Learning Research},
  publisher = 	 {PMLR},
  pdf = 	 {http://proceedings.mlr.press/v70/franceschi17a/franceschi17a.pdf},
}
```

For the work on learning-to-learn


```latex
@article{franceschi2017bridge,
  title={A Bridge Between Hyperparameter Optimization and Larning-to-learn},
  author={Franceschi, Luca and Frasconi, Paolo and Donini, Michele and Pontil, Massimiliano},
  journal={arXiv preprint arXiv:1712.06283},
  year={2017}
}
```
