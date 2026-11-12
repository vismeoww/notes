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
y_2 = \sigma(W_2x + b_2)
$$