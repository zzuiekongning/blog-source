---
title: Simple LSTM
date: 2018-09-28 10:04:47
tags:
---
A few weeks ago I released some [code](https://github.com/nicodjimenez/lstm) on Github to help people understand how LSTM's work at the implementation level.  The forward pass is well explained elsewhere and is straightforward to understand, but I derived the backprop equations myself and the backprop code came without any explanation whatsoever.  The goal of this post is to explain the so called *backpropagation through time* in the context of LSTM's. 

If you feel like anything is confusing, please post a comment below or submit an issue on Github.  

<!--more-->

*Note:* this post assumes you understand the forward pass of an LSTM network, as this part is relatively simple.  Please read this [great intro paper](http://arxiv.org/abs/1506.00019) if you are not familiar with this,  as it contains a very nice intro to LSTM's.  I follow the same notation as this paper so I recommend reading having the tutorial open in a separate browser tab for easy reference while reading this post.

# Introduction 
The forward pass of an LSTM node is defined as follows:

$$\begin{eqnarray}
g(t) &=& \phi(W_{gx} x(t) + W_{gh} h(t-1) + b_{g}) \\ 
i(t) &=& \sigma(W_{ix} x(t) + W_{ih} h(t-1) + b_{i}) \\ 
f(t) &=& \sigma(W_{fx} x(t) + W_{fh} h(t-1) + b_{f}) \\ 
o(t) &=& \sigma(W_{ox} x(t) + W_{oh} h(t-1) + b_{o}) \\ 
s(t) &=& g(t) * i(t) + s(t-1) * f(t)  \\ 
h(t) &=& s(t) * o(t) \\ 
\end{eqnarray}$$

By concatenating the $x(t)$ and $h(t-1)$ vectors as follows: 

$$x_c(t) = [x(t), h(t-1)]$$ 

we can re-write parts of the above as follows:

$$\begin{eqnarray}
g(t) &=& \phi(W_{g} x_c(t) + b_{g}) \\ 
i(t) &=& \sigma(W_{i} x_c(t) + b_{i}) \\ 
f(t) &=& \sigma(W_{f} x_c(t) + b_{f}) \\ 
o(t) &=& \sigma(W_{o} x_c(t) + b_{o}). \\
\end{eqnarray}$$

Suppose we have a loss $l(t)$ that we wish to minimize at every time step $t$ that depends on the hidden layer $h$ and the label $y$ at the current time via a loss function $f$: 

$$
l(t) = f(h(t), y(t))
$$

where $f$ can be any differentiable loss function, such as the Euclidean loss:

$$
l(t) = f(h(t), y(t)) = \| h(t) - y(t) \|^2.
$$

Our ultimate goal in this case is to use gradient descent to minimize the loss $L$ over an entire sequence of length $T$:

$$
L = \sum_{t=1}^{T} l(t).
$$

Let's work through the algebra of computing the loss gradient:

$$  
\frac{dL}{dw}
$$

where $w$ is a scalar parameter of the model (for example it may be an entry in the matrix $W_{gx}$).  Since the loss $l(t) = f(h(t), y(t))$ only depends on the values of the hidden layer $h(t)$ and the label $y(t)$, we have by the chain rule:

$$
\frac{dL}{dw} = \sum_{t = 1}^{T} \sum_{i = 1}^{M} \frac{dL}{dh_i(t)}\frac{dh_i(t)}{dw} 
$$

where $h_i(t)$ is the scalar corresponding to the $i$'th memory cell's hidden output and $M$ is the total number of memory cells.  Since the network propagates information forwards in time, changing $h_i(t)$ will have no effect on the loss prior to time $t$, which allows us to write:

$$\begin{eqnarray}
\frac{dL}{dh_i(t)} &=& \sum_{s=1}^T \frac{dl(s)}{dh_i(t)} = \sum_{s=t}^T \frac{dl(s)}{dh_i(t)}
\end{eqnarray}
$$

For notational convenience we introduce the variable $L(t)$ that represents the cumulative loss from step $t$ onwards:

$$
L(t) = \sum_{s=t}^{s=T} l(s)
$$

such that $L(1)$ is the loss for the entire sequence.  This allows us to re-write the above equation as:

$$
\frac{dL}{dh_i(t)} = \sum_{s=t}^T \frac{dl(s)}{dh_i(t)} = \frac{dL(t)}{dh_i(t)}
$$

With this in mind, we can re-write our gradient calculation as:

$$
\frac{dL}{dw} = \sum_{t = 1}^{T} \sum_{i = 1}^{M} \frac{dL(t)}{dh_i(t)}\frac{dh_i(t)}{dw} 
$$

Make sure you understand this last equation.  The computation of $\frac{dh_i(t)}{dw}$ follows directly follows from the forward propagation equations presented earlier.  We now show how to compute $\frac{dL(t)}{dh_i(t)}$ which is where the so called *backpropagation through time* comes into play.  

# Backpropagation through time

This variable $L(t)$ allows us to express the following recursion:

$$
L(t) = \begin{cases} l(t) + L(t+1) & \text{if} \, t < T \\ 
l(t) & \text{if} \, t = T
\end{cases}
$$

Hence, given activation $h(t)$ of an LSTM node at time $t$, we have that:

$$
\frac{dL(t)}{dh(t)} = \frac{dl(t)}{dh(t)} + \frac{dL(t+1)}{dh(t)}
$$

Now, we know where the first term on the right hand side $ \frac{dl(t)}{dh(t)}$ comes from: it's simply the elementwise derivative of the loss $l(t)$ with respect to the activations $h(t)$ at time $t$.  The second term $ \frac{dL(t+1)}{dh(t)} $ is where the recurrent nature of LSTM's shows up.  It shows that the we need the *next* node's derivative information in order to compute the current *current* node's derivative information.  Since we will ultimately need to compute $\frac{dL(t)}{dh(t)}$ for all $t=1,\dots,T$, we start by computing 

$$
\frac{dL(T)}{dh(T)} = \frac{dl(T)}{dh(T)} 
$$ 

and work our way backwards through the network. Hence the term *backpropagation through time*.  With these intuitions in place, we jump into the code.  

# Code 

We now present the code that performs the backprop pass through a single node at time $1 \leq t \leq T$.  The code takes as input: 

  * `top_diff_h` $= \frac{dL(t)}{dh(t)} = \frac{dl(t)}{dh(t)} + \frac{dL(t+1)}{dh(t)}$
  * `top_diff_s` $= \frac{dL(t+1)}{ds(t)}$.

And computes: 

  * `self.state.bottom_diff_s`  $= \frac{dL(t)}{ds(t)}$
  * `self.state.bottom_diff_h`  $= \frac{dL(t)}{dh(t-1)}$

whose values will need to be propagated backwards in time.  The code also adds derivatives to:

  * `self.param.wi_diff` $= \frac{dL}{dW_{i}}$
  * ... 
  * `self.param.bi_diff` $= \frac{dL}{db_{i}}$
  * ...

since recall that we must sum the derivatives from each time step: 

$$
\frac{dL}{dw} = \sum_{t = 1}^{T} \sum_{i = 1}^{M} \frac{dL(t)}{dh_i(t)}\frac{dh_i(t)}{dw}.
$$

Also, note that we use:

  * `dxc` $= \frac{dL}{dx_c(t)}$

where we recall that $x_c(t) = [x(t), h(t-1)]$.  Without any further due, the code:

{% codeblock %}
def top_diff_is(self, top_diff_h, top_diff_s):
    # notice that top_diff_s is carried along the constant error carousel
    ds = self.state.o * top_diff_h + top_diff_s
    do = self.state.s * top_diff_h
    di = self.state.g * ds
    dg = self.state.i * ds
    df = self.s_prev * ds

    # diffs w.r.t. vector inside sigma / tanh function
    di_input = (1. - self.state.i) * self.state.i * di 
    df_input = (1. - self.state.f) * self.state.f * df 
    do_input = (1. - self.state.o) * self.state.o * do 
    dg_input = (1. - self.state.g ** 2) * dg

    # diffs w.r.t. inputs
    self.param.wi_diff += np.outer(di_input, self.xc)
    self.param.wf_diff += np.outer(df_input, self.xc)
    self.param.wo_diff += np.outer(do_input, self.xc)
    self.param.wg_diff += np.outer(dg_input, self.xc)
    self.param.bi_diff += di_input
    self.param.bf_diff += df_input       
    self.param.bo_diff += do_input
    self.param.bg_diff += dg_input       

    # compute bottom diff
    dxc = np.zeros_like(self.xc)
    dxc += np.dot(self.param.wi.T, di_input)
    dxc += np.dot(self.param.wf.T, df_input)
    dxc += np.dot(self.param.wo.T, do_input)
    dxc += np.dot(self.param.wg.T, dg_input)

    # save bottom diffs
    self.state.bottom_diff_s = ds * self.state.f
    self.state.bottom_diff_x = dxc[:self.param.x_dim]
    self.state.bottom_diff_h = dxc[self.param.x_dim:]
{% endcodeblock %}

# Details 
The forward propagation equations show that <font color=#00fff>modifying $s(t)$ affects the loss $L(t)$ by directly changing the values of $h(t)$ as well as $h(t+1)$.  However, modifying $s(t)$ affects $L(t+1)$ only by modifying $h(t+1)$</font>.  Therefore, by the chain rule: 

$$
\begin{eqnarray}
\frac{dL(t)}{ds_i(t)} &=& \frac{dL(t)}{dh_i(t)} \frac{dh_i(t)}{ds_i(t)} + \frac{dL(t)}{dh_i(t+1)} \frac{dh_i(t+1)}{ds_i(t)}  \\
&=& \frac{dL(t)}{dh_i(t)} \frac{dh_i(t)}{ds_i(t)} + \frac{dL(t+1)}{dh_i(t+1)} \frac{dh_i(t+1)}{ds_i(t)}  \\
&=& \frac{dL(t)}{dh_i(t)} \frac{dh_i(t)}{ds_i(t)} + \frac{dL(t+1)}{ds_i(t)}  \\
&=& \frac{dL(t)}{dh_i(t)} \frac{dh_i(t)}{ds_i(t)} + [\texttt{top_diff_s}]_i.  \\
\end{eqnarray}
$$

Since the forward propagation equations state:

$$
h(t) = s(t) * o(t)
$$

we get that:

$$
\frac{dL(t)}{dh_i(t)} \frac{dh_i(t)}{ds_i(t)} =  o_i(t) * [\texttt{top_diff_h}]_i.
$$

Putting all this together we have:

```
ds = self.state.o * top_diff_h + top_diff_s
```

The rest of the equations should be straightforward to derive, please let me know if anything is unclear.  

Make sure to check out [Mathpix](https://mathpix.com) if you are a LaTeX user and [Losswise](https://losswise.com) if you need a monitoring solution for training deep learning models.

Happy training!