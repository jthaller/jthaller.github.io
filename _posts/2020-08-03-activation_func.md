---
title: "Activation Functions Review"
date: '2020-08-22'
categories:
  - miscellaneous
tags:
  - NN
---

## Sigmoid

### Things to note:

$$
y = \frac{1}{1+e^{-x}}
$$

- Max is 1, occurs around $$x=4$$. Min is zero, occurs around $$x=-4$$. Not a great range. When $$x=0$$, $$y=0.5$$. Sigmoid works like a probability since it returns a number between 0 and 1.

![Sigmoid](https://upload.wikimedia.org/wikipedia/commons/thumb/8/88/Logistic-curve.svg/1200px-Logistic-curve.svg.png)

## Tanh vs. sigmoid

![tanh](https://miro.medium.com/max/1190/1*f9erByySVjTjohfFdNkJYQ.jpeg)

### Things to note:

- Max is 1 but min is -1. At $$z=0$$, $$y=0$$. Grows faster than the conventional sigmoid. Note, sigmoid is just a shape. Technically tanh is also a sigmoid function, but data scientists mean the first equation when they say sigmoid.

## Linear activation

- Not really an activation. It's what it sounds like. $$y=w*x + b$$. Might show up on the exam.

## ReLU

$$
Max(0,x)
$$

![relu](https://www.researchgate.net/profile/Muhammad_Hamdan9/publication/327435257/figure/fig4/AS:742898131812354@1554132125449/Activation-Functions-ReLU-Tanh-Sigmoid.ppm)

### Things to note:

- No saturation problem like sigmoids, but it can have the issue of a dying-relu, meaning that when the learning rate is too high, or when a neuron learns a large negative bias term for its weights, the ReLu will be stuck yielding zero all the time since it only takes negative inputs. That neural becomes is thus useless.

## Leaky ReLU

$$
Max(.1x, x)
$$

![leaky](https://i1.wp.com/clay-atlas.com/wp-content/uploads/2019/10/image-37.png?w=640&ssl=1)

### Things to Note:

- There is also a version called parametric relu, which is $$max(\alpha x,x)$$, where $$\alpha$$ is a learnable parameter. I think this is the same $$\alpha$$ as the slope in gradient descent, but I'm not positive.
