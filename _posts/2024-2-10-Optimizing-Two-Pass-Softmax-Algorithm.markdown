---
layout: post
title:  "Optimizing Two-Pass Softmax Algorithm"
date:   2024-2-10 10:46:39 -0700
# categories: jekyll update
---

In 2020, Dukhan and Ablavatski introduced an efficient two-pass softmax algorithm that significantly enhanced the performance of inference engines. This innovation undoubtedly saves considerable computational resources.

<p align="center">
  <img src="/assets/two-pass-softmax.png" alt="Two Pass Softmax" style="width:50%;"/>
</p>
<p align="center">
  Two Pass Softmax Algorithm
</p>
\
It's worth noting that operations based on **base 2** can also be performed using **base e**. The process, which involves extracting the exponent and reconstructing the number with its exponent and mantissa, can be greatly simplified. Specifically, extracting the exponent becomes straightforward, and constructing the number using the exponent and mantissa boils down to a simple exponential (**exp**) calculation. Below is the revised version of the two-pass softmax algorithm, adapted to **base e**.

```python
import numpy as np

N = 100
X = np.random.rand(N)
Y = np.zeros(N)

m_sum = 0
n_sum = -np.Infinity

# 1st Pass
for xi in X:
  n_max = max(n_sum, xi)
  exp_xi = np.exp(xi - n_max)
  m_sum = exp_xi + m_sum * np.exp(n_sum - n_max)
  n_sum = n_max

norm = 1 / m_sum

# 2nd Pass
for i in range(N):
  exp_xi = np.exp(X[i] - n_sum)
  Y[i] = exp_xi * norm


# Reference softmax implementation for comparison
def softmax(x):
    e_x = np.exp(x - np.max(x))
    return e_x / e_x.sum()

np.allclose(softmax(X), Y)
```

