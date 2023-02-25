---
title: "Let's Talk Transfer Learning"
date: '2021-07-30'
categories:
  - miscellaneous
tags: 
- transfer-learning
---

One-shot transfer learning with regression

## Motivation
Super quick project recap: I want to predict the disorder (label) from a spectrum (signal). I'd like to make the prediction on experimental data, but there are only a few examples. So I trained the network on simulation data and am now applying transfer learning.

Here's a nice figure I made this week and a helpful reminder for what the spectra I'm working with look like, in general.
<!-- <iframe src="/assets/images/same_r_random_dir_plotly.html" height="600px" width="150%" style="border:none;"></iframe> -->
![](https://github.com/jthaller/jthaller.github.io/blob/master/assets/images/interpolation-quality-exp-data.png?raw=true)

## Approach and what I've been thinking about this week
One-shot learning with regression is a difficult problem to say the least. With images, you can show a picture of a dog to a baby that's never seen one before, and the baby will now be able to identify all dogs, just from having seen that one photo. This is an example of one-shot learning. In practice, it is a difficult task to achieve one-shot learning classification on a neural network, but because we can do it, it seems like a somewhat reasonable goal. 

One-shot learning regression problems, however, do not have quite as clean an analogy. The closest example I can think of is if you showed a plot $$sin(\omega t)$$ for a few different values of $$\omega$$. A person could probably make a guess for a new sin wave's period based on which two functions it looked most similar to. But any problem even remotely more complicated becomes impossible for humans. In the case of my neural network, my model doesn't have just one parameter like $$\omega$$, my model requires 100K+ parameters and many convolutional layers just to be able to learn the complex patterns encoded in the simulation spectra.

So, how does one go about applying transfer learning, anyway? A lot of common approaches for classification don't work well with regression, and even the ones that do, still require more data than I have. For example, Siamese networks are a common way for training a network for classifying fraudulent signatures that can be expanded with one-shot learning pretty well. The idea is to train two identical networks (both architecture and parameters) at the same time, but provide different inputs to each network. You create a pairs of training samples and one half of the pair to one network and the other half to the other network. The network learns to predict the difference between real and fraudulent signatures based on the output of the networks for valid-fraudulent pairs.

>FREE PAPER IDEA: I think you could train a Siamese CNN to remove noise from absorption spectra (or really any signal processing) by creating a matching synthetic dataset with Gaussian white noise added to the signals.

So if Siamese NN's are so great, why won't this work for my project? Well, I don't have pairs of simulation-experimental data. I want the network to systematically learn the difference between the simulation and experimental data, but I only have 2 examples of experimental data---one of which I need to save for testing! 

One approach for few-shot regression was published in 2017 from the University of Singapore. They reduced the latent space of the problem by Fourier expanding the functions and then predicting the coefficients of the components. So long as the number of training examples is greater than the number of coefficients needed to model the function, training will work well. I like this approach, but I'm not convinced I have enough data for this to work well on my project.

I don't want to divulge too much about my approach, because...you know...unpublished work...but I'm essentially relying on several stages transfer learning, data-augmentation (synthetic experimental datasets), and purposeful injection of bias early on in the process to select candidates likely to perform well for the fine-tuning process.




