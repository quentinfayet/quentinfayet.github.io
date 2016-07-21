---
layout: page
title: A dive into Neural Networks
comments: false
modified: 2016-07-19
tags: ["machine learning", "neural network", "ai", "deep learning"]
---

Neural Networks are like the dark side of the Machine Learning. Some people
love them (I do), but some don't understand the underlying principles, and thus,
come to fear them.

What could be more exciting than mimic the human (animal) brain? I mean, if we
simplify the principle of the brain at its maximum, it is nothing else than
electric impulses sailing through neurons.

Talking this way makes it simple to mimic. We could easily imagine a imbricated
structure that would propagate some kind of signal. In our case, it would be
digital signal, but in the end, hardware speaking, it remains nothing but
electric signal

Although it is true that nowadays, some principle that make Neural Networks work
just fine are still blur, our global comprehension has never been so great.

Using Neural Networks, we can perform signal processing at large scale, perform
pattern recognition over huge amount of data, or even generate text, pictures,
and even [movies](https://github.com/graphific/DeepDreamVideo).

Who hasn't heard about Google's Go Artificial Intelligence player, DeepMind
Or DeepDream, the "dreaming" neural network, also by Google? And who know
what will come out of Google's Google Brain division in the months to come?

In this article, I propose to take you in a tour of neural networks, their history,
their utility, and the way they work.

Fasten your seatbelt, let's go back in time! (yes, neural network is an old idea!).

## Back in the 40's

In 1943, two scientists, neurophysiologist Warren McCulloch and mathematician
studied the way biological neurons might work. They simplified the process by
modelizing biological networks with electrical circuits. They've published their
work in a paper, [*A logical calculus of the ideas immanent in nervous activity*](http://vordenker.de/ggphilosophy/mcculloch_a-logical-calculus.pdf),
and by the way created the first ever logical, electrical artificial neuron. This
model of neuron, called as MCP neuron, is still in use nowadays, and I'll talk
about it again later.

A few years later in 1949, another scientist, Donald Hebb, a psychologist, wrote
an other paper, [*The organization of behavior*](http://deeplearning.cs.cmu.edu/pdfs/Hebb.1949.pdf),
in which he assumes the fact that pathways between neurons are getting stronger
every time they're used. That would comes to explain how human can learn.

Years later, as computers were getting more and more powerful, Bernard Widrow a
professor in electrical engineering and his then doctoral student Ted Hoff
invented an algorithm known as "Widdrow-Hoff LMS" (Least Mean Squares)
