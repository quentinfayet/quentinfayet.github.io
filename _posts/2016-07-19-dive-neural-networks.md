---
layout: page
title: A dive into Neural Networks
comments: false
modified: 2016-07-19
tags: ["machine learning", "neural network", "ai", "deep learning"]
---

## Coucou

{% latex density=300 resize=1000x1000 usepackages=tikz %}
\def\layersep{2.5cm}

\begin{tikzpicture}[shorten >=1pt,->,draw=black!50, node distance=\layersep]
    \tikzstyle{every pin edge}=[,shorten <=1pt]
    \tikzstyle{neuron}=[circle,fill=black!25,minimum size=17pt,inner sep=0pt]
    \tikzstyle{input variable}=[neuron, fill=green!50];
    \tikzstyle{sum function}=[neuron, minimum size=30pt];
    \tikzstyle{activation function}=[neuron, fill=red!50, minimum size=30pt];
    \tikzstyle{weight}=[neuron, fill=blue!50];
    \tikzstyle{annot} = [text width=4em, text centered]

    % Draw the input layer nodes
    \foreach \name / \y in {1,...,4}
    % This is the same as writing \foreach \name / \y in {1/1,2/2,3/3,4/4}
        \node[input variable] (V-\name) at (0,{-(1 + \y)}) {$x_\y$};

    % Draw the hidden layer nodes
    \node[weight, label=left:bias] (W-0) at (\layersep,0) {$\theta_0$};

    \foreach \name / \y in {1,...,4}
        \path
            node[weight] (W-\name) at (\layersep,{-(1 + \y)}) {$\theta_\y$};

    % Draw the output layer node
    \node[sum function, right of=W-3] (S) {$\sum$};

    \node[activation function, pin={[pin edge={->}]right:y (ouput)}, right of=S] (A) {$\varphi$};

    % Connect every node in the input layer with every node in the
    % hidden layer.
    \foreach \variable in {1,...,4}
            \path (V-\variable) edge (W-\variable);

    % Connect every node in the hidden layer with the output layer
    \foreach \weight in {0,...,4}
        \path (W-\weight) edge (S);

    \path (S) edge (A);

    % Annotate the layers
    \node[annot,above of=W-0, node distance=1cm] (weights) {Weights};
    \node[annot,left of=weights] {Input variables};
    \node[annot,right of=weights] (sf) {Sum function};
    \node[annot,right of=sf] {Activation function};
\end{tikzpicture}
{% endlatex %}
