> [!warning] Work in Progress: This post is currently incomplete and still being drafted. Check back soon for updates!

I have been spending some time in learning how machine learning libraries such as `pytorch` work. Something that I kept coming across while learning about it is the concept of `autograd` 
To explain the concept of autograd, I will start from scratch for how deep neural networks function. 
# Single layer of Neural Network
Let's consider the following layer for a simple neural network. There are 2 input variables and 3 neurons, each of them has an add bias and then followed by an activation functions

![[BasicNN.excalidraw]]

This can also be written using the mathematical equation 
$$
y = \sigma(Wx + b)
$$
where $W$ is the weight matrix, $x$ is the input, $b$ is the bias and $\sigma$ is the activation function.
if there are two layers to the neural network then we will write 
$$
y_1 = \sigma(W_1x + b_1)\\
y_2 = \sigma(W_2y_1 + b_2)
$$

## Loss and calculating gradient

Okay let's come back to the single layer neural network, if you look at the equation again, we can see that there are two parameters inside it; namely $W$ and $b$ . These are values that can be modified to move the neural network towards producing the correct answer. The so called gradient descent algorithm lets us do that. 

But how does moving these two parameters help us train the neural network you may ask? Let's see what exactly it means to train a neural network. 

For any input $x$ let's say we have an expected output called target or $t$  and the value of $y$ is what our neural network predicted the target should be for the corresponding $x$ 
So let's define something called a `loss function`
$$
Loss = \frac{\sum_{i=0}^{N} (y_i - t_i)^2}{N}
$$
where $y_i,t_t$ are the $i^{th}$ element of our output vector and target vector respectively and $N$ is the number of elements in the vector.
For sake of simplicity let's represent `loss` as 
$$
Loss = (y - t)^2
$$
Now for the neural network to produce correct predictions, the loss should come down. 
Let's see how that's done. 

## Gradient Descent
Before starting with gradient descent, let's start with something rather simple. Let's say we have a function $y = f(x)$ , and we need to find where the function is at it's smallest value. let's assume that position is at $x_1$ , this allows us to make some statements. We can see that 
$$
f(x_1+\epsilon) > f(x_1)\\
f(x_1-\epsilon) > f(x_1)
$$
or we can also say that it's gradient at $x_1$
$$
\frac{d}{dx}(f(x))|x=x_1 = 0 \hspace{1cm} or \hspace{1cm} f'(x_1) = 0
$$
