---
title: "Entropy, Information, and Guess Who"
date: '2020-11-09'
categories:
  - miscellaneous
tags:
  - Math
  - Decision Trees
---
<div style="text-align: justify">
Decision trees are one of the first data structures you learn in computer science (graphically, in practice you might learn them as a linked list), but you might not have learned the best way to construct a decision tree.

Of course there are several ways to decide how to grow a decision tree, but the most common way is to order the splits by the amount of information gained. To quantify the information gained we need to consider the entropy change between layers. If you're a [former] physicist like me, you'll recall that entropy is defined as:

$$
S = k \ln{\Omega}
$$

where k is Boltzmann's constant and $$\Omega$$ is the number of unique configurations of the system. For example, for 10 unique cards, there are $$\Omega = 10!$$ configurations. In computer science, however, we define entropy in the same form, but with different, relevant parameters:

$$
S\left[\text{Node Split 1}\right] = -\sum_{i=1}^n p_i \log{p_i}
$$

I've switched from ln to log, but I mean the same for both. I'm switching from physics to CS convention. Here, $$p_i$$ is the fraction of a given label for the tree split. Let's give a concrete example by talking about Guess Who.

## Guess Who

![](https://www.geekyhobbies.com/wp-content/uploads/2016/02/Guess-Who-1.jpg)

Mark Rober made a great YouTube video about the optimal strategy for playing Guess Who. Much to my satisfaction, it was employed the same strategy that I had tried as a child (much to the annoyance of the rest of my family). He strategy was to combine features into categories that would be able to split the remaining characters in half after each round's question. Before we turn this into a decision tree, let's start by looking a simple case.

Since we're interested in the change in entropy, we need a starting entropy value. In the game, there are 24 characters total with one being chosen as the target person. Okay, technically, since it's a two person game and your opponent can't have the same chosen person as you, there are only 23 options, but I'll ignore that for simplicity. ***Let's classify Bill as the correct guess and all others as incorrect***. Thus, the initial entropy is:

$$
S_0 = \underbrace{-\frac{1}{24}\log\left(\frac{1}{24}\right)}_\text{Correct Guess} - \underbrace{\frac{23}{24}\log\left(\frac{23}{24}\right)}_\text{Incorrect Guesses} \approx 0.173
$$

Now we want to know how this entropy changes when you choose a feature split for the first level. Most people's first move is to ask "Is your person a man or a woman?" (or whatever the gender inclusive version of this is. There's actually a lot of ways the original game is a social nightmare and pretty terrible, but that's not what I blog about). So, what's the entropy of this?

$$
S[\text{Man}] = \underbrace{-\frac{1}{19}\log\left(\frac{1}{19}\right)}_\text{Correct Guess} - \underbrace{\frac{18}{19}\log\left(\frac{18}{19}\right)}_\text{Incorrect Guesses} \approx 0.206
$$

Now, for women:

$$
S[\text{Women}] = \underbrace{-\frac{0}{5}\log\left(\frac{0}{5}\right)}_\text{Incorrect Guesses} = 0
$$

Typically, in a decision tree you might you'd have multiple "Trues" and "Falses" classified under each split, but we're talking about Guess Who. So how much information is gained through the addition of this tree split? It's the original entropy minus the weighted change of entropy for each branch of the split.

$$
\begin{aligned}
\text{Information Gained} =&~ S_0 -S[\text{Original Tree} \mid \text{Man/Woman Split}]
\\
=&~ S_0 - \frac{19}{24}S[\text{Man}] - \frac{5}{24}S[\text{Woman}]
\\
=&~ 0.173 - \frac{19}{24} * 0.206 - \frac{5}{24}*0 \approx 0.00991
\end{aligned}
$$

This is the information gained if you ask whether your opponent has a man and they say yes. The your opponent had a woman, we'd expect a greater amount of information gain, right? In this case, S[Man] and S[Wom] would be different, and the information gained would be:

$$
\text{Information Gained} = 0.173 - \frac{19}{24} * 0 - \frac{5}{24}* \left\{ -\frac{1}{5}\log\left( \frac{1}{5}\right) - \frac{4}{5}\log\left(\frac{1}{5}\right) \right\} \approx 0.06895
  $$

See how much more information is gained from if you discover that your opponent picked a woman?

## Splitting trees

In Guess Who, there are many more questions to ask, each of which will give you a certain amount of information. The order in which a decision tree is made is based on which question gives the most information. Each subsequent node split in the tree is a question that provides a smaller information gain. In Guess Who, the question "*Is your person Bill?*" would give the most information if that were indeed the correct guess. If it weren't, it would be a pretty lousy question to start out with.

The game, thus, requires a balance between information gain and the probability that the question you ask will give you that information. This is why Mark Rober's approach was to try and remove as much probability from the game as possible, and stick to True/False questions that split the possibilities in half regardless of how the question is answered.

</div>

## Tree Impurities

Three common ways to measure tree impurities are misclassification error, the Gini Index, and cross-entropy error. The Gini index and cross-entropy more common because they are more sensitive to changes in node probabilities

1. Misclassification error:

$$
Error = 1 - Argmax(fraction~true, fraction~false)
$$

Say the split has 60% of outcomes on the left and 40% on the right. Of the left outcomes, 100% are True, and of the right outcomes 30% are true. Then:

$$
Error~Left = 1 - Argmax(100 \%, 0\%) \\
Error~Right = 1 - Argmax(30\%, 70\%) \\
\Delta Error = E_{orig} - .4*1 - .6*.7
$$

2. The Gini Index is bounded between 0 and 1/2. The lower the value, the better a split is, since it indicates that probability of classifying it wrong. You always want to choose splits (top down) starting with the split with the lower Gini Index. 

$$
\text{Unweighted Gini Index} = 2P_1 P_2 \\
\text{Weighted Gini Index} = P_{left} 2P_1 P_2 + P_{right}2P_1 P_2
$$

The Gini Index doesn't really make sense to use in the context of Guess Who because the two probabilities are multiplied and will always be zero. To clarify, I'll give a specific example. Let's say our person is Bill, and we are considering two splits: Man/Woman and Red hair/not red hair. As we know, there are 19 men and 5 women. Thus, P1 = 1/19 and P2 = 0/5. So the weighted Gini Index is:

$$
\text{Weighted Gini Index} = \left(\frac{19}{24}\right)2\left(\frac{1}{19}\right)\left(\frac{0}{5}\right) + \left(\frac{5}{24}\right)2\left(\frac{1}{19}\right)\left(\frac{0}{5}\right)
$$

See, pretty stupid in this context. But, you can imagine if you were using a decision tree with multiple Trues and Falses as outcomes, the Gini Index would be nonzero. You could calculate the index or other splits (such as red/not red hair) and the one with a lower index will be the better split to choose first.

3. I already talked about minimizing cross entropy in the previous section. When you minimize cross entropy, you maximize information gain.
</div>