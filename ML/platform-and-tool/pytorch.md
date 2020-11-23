[toc]

# 示例

## [Learning PyTorch with Examples](https://pytorch.org/tutorials/beginner/pytorch_with_examples.html#)

首先使用numpy实现一个简单的二层FNN：

```python
import numpy as np

# N is batch size; D_in is input dimension;
# H is hidden dimension; D_out is output dimension.
N, D_in, H, D_out = 64, 1000, 100, 10

# Create random input and output data
x = np.random.randn(N, D_in)
y = np.random.randn(N, D_out)

# Randomly initialize weights
w1 = np.random.randn(D_in, H)
w2 = np.random.randn(H, D_out)

learning_rate = 1e-6
for t in range(500):
    # Forward pass: compute predicted y
    h = x.dot(w1)
    h_relu = np.maximum(h, 0)
    y_pred = h_relu.dot(w2)

    # Compute and print loss
    loss = np.square(y_pred - y).sum()
    if t % 10 == 0:
      print(t, loss)

    # 计算loss对w1,w2的梯度,需要手动输入公式
    grad_y_pred = 2.0 * (y_pred - y)
    grad_w2 = h_relu.T.dot(grad_y_pred)
    grad_h_relu = grad_y_pred.dot(w2.T)
    grad_h = grad_h_relu.copy()
    grad_h[h < 0] = 0
    grad_w1 = x.T.dot(grad_h)

    # Update weights
    w1 -= learning_rate * grad_w1
    w2 -= learning_rate * grad_w2
```

现在用torch实现上述FNN，并且使用自动梯度计算autograd：

```python
import torch

# N is batch size; D_in is input dimension;
# H is hidden dimension; D_out is output dimension.
N, D_in, H, D_out = 64, 1000, 100, 10

# 创建随机的输入和输出向量
x = torch.randn(N, D_in)
y = torch.randn(N, D_out)

# 创建随机的权重向量
# 参数 requires_grad=True 表示希望在backward pass过程中计算对于这些张量的梯度
w1 = torch.randn(D_in, H, requires_grad=True)
w2 = torch.randn(H, D_out, requires_grad=True)

learning_rate = 1e-6
for t in range(500):
    # Forward pass: 使用x(输入向量)和网络结构计算(张量操作)出y(输出向量)
    y_pred = x.mm(w1).clamp(min=0).mm(w2)

    # 计算损失
    # loss 是 (1,) 形状的张量,故使用 loss.item() 获取其中的标量值
    loss = (y_pred - y).pow(2).sum()
    if t % 100 == 99:
        print(t, loss.item())

    # 使用autograd计算backward pass过程,这将计算loss对于所有参数为
    # requires_grad=True 的向量的梯度.调用结束后梯度计算结果将保存在
    # w1.grad 和 w2.grad 中.
    loss.backward()

    # 手动更新梯度.
    # Wrap in torch.no_grad() because weights have requires_grad=True, 
    # but we don't need to track this in autograd.
    # An alternative way is to operate on weight.data and weight.grad.data.
    # Recall that tensor.data gives a tensor that shares the storage with
    # tensor, but doesn't track history.
    # 使用 torch.optim.SGD 可以达到同样的效果.
    with torch.no_grad():
        w1 -= learning_rate * w1.grad
        w2 -= learning_rate * w2.grad

        # 手动在更新权重之后将梯度置零
        w1.grad.zero_()
        w2.grad.zero_()
```

`torch.nn`提供了更高级的抽象（相当于Tensorflow的keras），帮助我们更便捷地搭建网络。使用`nn`再次实现上述FNN：

```python
import torch

N, D_in, H, D_out = 64, 1000, 100, 10

x = torch.randn(N, D_in)
y = torch.randn(N, D_out)

# 使用 nn 包定义模型,nn.Sequential包含了一个层的序列,按照顺序依次执行
# 各种类型的层和nn.Sequential都是Module对象
model = torch.nn.Sequential(
    torch.nn.Linear(D_in, H),
    torch.nn.ReLU(),
    torch.nn.Linear(H, D_out),
)
# 使用 nn 包定义损失函数,这里使用MSE
loss_fn = torch.nn.MSELoss(reduction='sum')

# 使用 optim 包定义一个优化器来为我们的模型更新权重参数,这里使用Adam
# 第一个参数为优化器需要去更新的参数,第二个参数为学习率
learning_rate = 1e-4
optimizer = torch.optim.Adam(model.parameters(), lr=learning_rate)
for t in range(500):
    # Forward pass: 将x输入模型得到y的预测结果
    # Module类重载了__call__方法因而可以像函数一样调用
    y_pred = model(x)

    # 输入y的预测值和实际值,由损失函数返回损失值
    loss = loss_fn(y_pred, y)
    if t % 10 == 0:
        print(t, loss.item())

    # 在backward pass之前,使用优化器归零(损失)对于需要更新的参数的梯度(也就是模型的权重参数)
    # 这在因为在调用 loss.backward() 时缓存区中的梯度会累积(而不是覆盖)
    optimizer.zero_grad()

    # Backward pass: 计算损失对于模型中所有可学习参数的梯度.因为每个Module中的参数都有
    # requires_grad=True
    loss.backward()

    # 调用优化器的step函数来更新一次参数
    optimizer.step()
```



## CNN

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# 定义神经网络模型类
class CNNnet(nn.Module):
    def __init__(self): # 构造函数必需,对象属性为各网络层
      super(Net, self).__init__()

      # 二维卷积层,输入1通道(灰度图片),输出32卷积特征/通道
      # 卷积核为3x3,步长为1
      self.conv1 = nn.Conv2d(1, 32, 3, 1)
      # 二维卷积层,输入32卷积特征/通道,输出64卷积特征/通道
      self.conv2 = nn.Conv2d(32, 64, 3, 1)

      # 丢弃层
      self.dropout1 = nn.Dropout2d(0.25)
      self.dropout2 = nn.Dropout2d(0.5)

      # 全连接层
      self.fc1 = nn.Linear(9216, 128)
      self.fc2 = nn.Linear(128, 10)

    def forward(self, x):
      print(x.shape)  # [100, 1, 28, 28]
    
      x = self.conv1(x)
      print(x.shape)  # [100, 32, 26, 26]
      # ReLU激活函数
      x = F.relu(x)
      print(x.shape)

      x = self.conv2(x)  # [100, 64, 24, 24]
      print(x.shape)
      x = F.relu(x)
      print(x.shape)

      # 最大汇聚
      x = F.max_pool2d(x, 2)  # [100, 64, 12, 12]
      print(x.shape)
      # Pass data through dropout1
      x = self.dropout1(x)
      print(x.shape)
      # Flatten x with start_dim=1
      x = torch.flatten(x, 1) # [100, 9216]
      print(x.shape)
      x = self.fc1(x)
      print(x.shape)          # [100, 128]
      x = F.relu(x)
      print(x.shape)
      x = self.dropout2(x)
      print(x.shape)          # [100, 10]
      x = self.fc2(x)
      print(x.shape)

      # softmax回归
      output = F.softmax(x, dim=1)
      print(output.shape)
      return output  

my_nn = CNNnet()
# 100张28x28灰度图片
random_data = torch.rand((100, 1, 28, 28))
# [N, C, H, W]
# N = batch_size = 100
# C = channel = 1
# H = height = 28
# W = width = 28
result = my_nn(random_data)
print(result)
```



## RNN



## 基于注意力的机器翻译

```python
# NLP From Scratch: Translation with a Sequence to Sequence Network and Attention

from __future__ import unicode_literals, print_function, division
from io import open
import unicodedata
import string
import re
import random

import torch
import torch.nn as nn
from torch import optim
import torch.nn.functional as F

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

# Loading data files
# The data for this project is a set of many thousands of English to
# French translation pairs.

SOS_token = 0
EOS_token = 1


class Lang:
    def __init__(self, name):
        self.name = name
        self.word2index = {}
        self.word2count = {}
        self.index2word = {0: "SOS", 1: "EOS"}
        self.n_words = 2  # Count SOS and EOS

    def addSentence(self, sentence):
        for word in sentence.split(' '):
            self.addWord(word)

    def addWord(self, word):
        if word not in self.word2index:
            self.word2index[word] = self.n_words
            self.word2count[word] = 1
            self.index2word[self.n_words] = word
            self.n_words += 1
        else:
            self.word2count[word] += 1


# The files are all in Unicode, to simplify we will turn Unicode
# characters to ASCII, make everything lowercase, and trim most
# punctuation.
# Turn a Unicode string to plain ASCII, thanks to
# https://stackoverflow.com/a/518232/2809427
def unicodeToAscii(s):
    return ''.join(c for c in unicodedata.normalize('NFD', s)
                   if unicodedata.category(c) != 'Mn')


# Lowercase, trim, and remove non-letter characters


def normalizeString(s):
    s = unicodeToAscii(s.lower().strip())
    s = re.sub(r"([.!?])", r" \1", s)
    s = re.sub(r"[^a-zA-Z.!?]+", r" ", s)
    return s


def readLangs(lang1, lang2, reverse=False):
    print("Reading lines...")

    # Read the file and split into lines
    lines = open('data/%s-%s.txt' % (lang1, lang2), encoding='utf-8').\
        read().strip().split('\n')

    # Split every line into pairs and normalize
    pairs = [[normalizeString(s) for s in l.split('\t')] for l in lines]

    # Reverse pairs, make Lang instances
    if reverse:
        pairs = [list(reversed(p)) for p in pairs]
        input_lang = Lang(lang2)
        output_lang = Lang(lang1)
    else:
        input_lang = Lang(lang1)
        output_lang = Lang(lang2)

    return input_lang, output_lang, pairs


MAX_LENGTH = 10

eng_prefixes = (
    "i am ", "i m ",
    "he is", "he s ",
    "she is", "she s ",
    "you are", "you re ",
    "we are", "we re ",
    "they are", "they re "
)

# filter sentences longer than 10 words
def filterPair(p):
    return len(p[0].split(' ')) < MAX_LENGTH and \
        len(p[1].split(' ')) < MAX_LENGTH and \
        p[1].startswith(eng_prefixes)             # reduce number of pairs to 10k+


def filterPairs(pairs):
    return [pair for pair in pairs if filterPair(pair)]


# The full process for preparing the data is:
# -  Read text file and split into lines, split lines into pairs
# -  Normalize text, filter by length and content
# -  Make word lists from sentences in pairs
def prepareData(lang1, lang2, reverse=False):
    input_lang, output_lang, pairs = readLangs(lang1, lang2, reverse)
    print("Read %s sentence pairs" % len(pairs))
    pairs = filterPairs(pairs)
    print("Trimmed to %s sentence pairs" % len(pairs))
    print("Counting words...")
    for pair in pairs:
        input_lang.addSentence(pair[0])
        output_lang.addSentence(pair[1])
    print("Counted words:")
    print(input_lang.name, input_lang.n_words)
    print(output_lang.name, output_lang.n_words)
    return input_lang, output_lang, pairs


input_lang, output_lang, pairs = prepareData('eng', 'fra', True)
print(random.choice(pairs))


# seq2seq model
# Encoder
class EncoderRNN(nn.Module):
    def __init__(self, input_size, hidden_size):
        super(EncoderRNN, self).__init__()
        self.hidden_size = hidden_size
        # embedding dimension = size of hidden state
        self.embedding = nn.Embedding(input_size, hidden_size)
        self.gru = nn.GRU(hidden_size, hidden_size)

    def forward(self, input, hidden):
        # input one index of word
        # output embedding tensor with shape [1, 1, 256]
        output = self.embedding(input).view(1, 1, -1)
        # only one GRU cell
        output, hidden = self.gru(output, hidden)
        return output, hidden

    def initHidden(self):
        return torch.zeros(1, 1, self.hidden_size, device=device)


# Simple Decoder, not used here
class DecoderRNN(nn.Module):
    def __init__(self, hidden_size, input_size):
        super(DecoderRNN, self).__init__()
        self.hidden_size = hidden_size

        self.embedding = nn.Embedding(input_size, hidden_size)
        self.gru = nn.GRU(hidden_size, hidden_size)
        self.out = nn.Linear(hidden_size,
                             input_size)  # map hidden state to index of word
        self.softmax = nn.LogSoftmax(dim=1)

    def forward(self, input, hidden):
        output = self.embedding(input).view(1, 1, -1)
        output = F.relu(output)
        output, hidden = self.gru(output, hidden)
        output = self.softmax(self.out(output[0]))
        return output, hidden

    def initHidden(self):
        return torch.zeros(1, 1, self.hidden_size, device=device)


# Decoder with attention mechanism
class AttnDecoderRNN(nn.Module):
    def __init__(self,
                 hidden_size,
                 output_size,
                 dropout_p=0.1,
                 max_length=MAX_LENGTH):
        super(AttnDecoderRNN, self).__init__()
        self.hidden_size = hidden_size
        self.output_size = output_size
        self.dropout_p = dropout_p
        self.max_length = max_length

        self.embedding = nn.Embedding(self.output_size, self.hidden_size)
        self.attn = nn.Linear(self.hidden_size * 2, self.max_length)
        self.attn_combine = nn.Linear(self.hidden_size * 2, self.hidden_size)
        self.dropout = nn.Dropout(self.dropout_p)
        self.gru = nn.GRU(self.hidden_size, self.hidden_size)
        self.out = nn.Linear(self.hidden_size, self.output_size)

    def forward(self, input, hidden, encoder_outputs):
        embedded = self.embedding(input).view(1, 1, -1)  # [1, 1, 256]
        embedded = self.dropout(embedded)

        attn_weights = F.softmax(
            self.attn(  # ??? # [1, 10]  
                torch.cat((embedded[0], hidden[0]), 1)),
            dim=1)
        attn_applied = torch.bmm(
            attn_weights.unsqueeze(0),  # [1, 1, 256] [1, 1, 10]
            encoder_outputs.unsqueeze(0))  # [1, 10 ,256]
        output = torch.cat((embedded[0], attn_applied[0]), 1)
        output = self.attn_combine(output).unsqueeze(0)  # [1, 1, 256]

        output = F.relu(output)
        output, hidden = self.gru(output,
                                  hidden)  # for one GRU cell, output==hidden

        output = F.log_softmax(self.out(output[0]), dim=1)
        return output, hidden, attn_weights

    def initHidden(self):
        return torch.zeros(1, 1, self.hidden_size, device=device)


# Preparing Training Data
def indexesFromSentence(lang, sentence):
    return [lang.word2index[word] for word in sentence.split(' ')]


def tensorFromSentence(lang, sentence):
    indexes = indexesFromSentence(lang, sentence)
    indexes.append(EOS_token)
    return torch.tensor(indexes, dtype=torch.long, device=device).view(-1, 1)


def tensorsFromPair(pair):
    input_tensor = tensorFromSentence(input_lang, pair[0])
    target_tensor = tensorFromSentence(output_lang, pair[1])
    return (input_tensor, target_tensor)


# Training the Model
#
# To train we run the input sentence through the encoder, and keep track
# of every output and the latest hidden state. Then the decoder is given
# the ``<SOS>`` token as its first input, and the last hidden state of the
# encoder as its first hidden state.
#
# "Teacher forcing" is the concept of using the real target outputs as
# each next input, instead of using the decoder's guess as the next input.
# Using teacher forcing causes it to converge faster but `when the trained
# network is exploited, it may exhibit
# instability <http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.378.4095&rep=rep1&type=pdf>`__.

use_teacher_forcing = True  # use teacher forcing or not


def train(
        input_tensor,  # train one pair of input&target tensor
        target_tensor,
        encoder,
        decoder,
        encoder_optimizer,
        decoder_optimizer,
        criterion,
        max_length=MAX_LENGTH):
    encoder_hidden = encoder.initHidden()  # zero tensor of shape [1, 1, 256]

    encoder_optimizer.zero_grad()
    decoder_optimizer.zero_grad()

    input_length = input_tensor.size(0)
    target_length = target_tensor.size(0)
    # example:
    # tensor([[ 350],
    #         [  72],
    #         [1031],
    #         [ 349],
    #         [ 131],
    #         [2248],
    #         [  14],
    #         [   1]])
    # input_length: 8
    # tensor([[  23],
    #         [ 230],
    #         [ 444],
    #         [  94],
    #         [1079],
    #         [   3],
    #         [   1]])
    # target_length: 7

    encoder_outputs = torch.zeros(max_length,
                                  encoder.hidden_size,
                                  device=device)

    loss = 0

    for ei in range(input_length):
        encoder_output, encoder_hidden = encoder(input_tensor[ei],
                                                 encoder_hidden)
        encoder_outputs[ei] = encoder_output[0, 0]

    decoder_input = torch.tensor([[SOS_token]], device=device)

    decoder_hidden = encoder_hidden

    if use_teacher_forcing:
        # Teacher forcing: Feed the target as the next input
        for di in range(target_length):
            decoder_output, decoder_hidden, decoder_attention = decoder(
                decoder_input, decoder_hidden, encoder_outputs)
            loss += criterion(decoder_output, target_tensor[di])

            decoder_input = target_tensor[di]  # Teacher forcing

    else:
        # Without teacher forcing: use its own predictions as the next input
        for di in range(target_length):
            decoder_output, decoder_hidden, decoder_attention = decoder(
                decoder_input, decoder_hidden, encoder_outputs)
            topv, topi = decoder_output.topk(1)
            decoder_input = topi.squeeze().detach(
            )  # detach from history as input

            loss += criterion(decoder_output, target_tensor[di])
            if decoder_input.item() == EOS_token:
                break

    loss.backward()

    encoder_optimizer.step()
    decoder_optimizer.step()

    return loss.item() / target_length


# print time elapsed and estimates time remaining given the current time and progress %.
import time
import math


def asMinutes(s):
    m = math.floor(s / 60)
    s -= m * 60
    return '%dm %ds' % (m, s)


def timeSince(since, percent):
    now = time.time()
    s = now - since
    es = s / (percent)
    rs = es - s
    return '%s (- %s)' % (asMinutes(s), asMinutes(rs))


# The whole training process looks like this:
#
# -  Start a timer
# -  Initialize optimizers and criterion
# -  Create set of training pairs
# -  Start empty losses array for plotting
#
# Then we call ``train`` many times and occasionally print the progress (%
# of examples, time so far, estimated time) and average loss.
#
def trainIters(encoder,
               decoder,
               n_iters,
               print_every=1000,
               plot_every=100,
               learning_rate=0.01):
    start = time.time()
    plot_losses = []
    print_loss_total = 0  # Reset every print_every
    plot_loss_total = 0  # Reset every plot_every

    encoder_optimizer = optim.SGD(encoder.parameters(), lr=learning_rate)
    decoder_optimizer = optim.SGD(decoder.parameters(), lr=learning_rate)
    training_pairs = [
        tensorsFromPair(random.choice(pairs)) for i in range(n_iters)
    ]
    criterion = nn.NLLLoss()

    for iter in range(1, n_iters + 1):
        training_pair = training_pairs[iter - 1]
        input_tensor = training_pair[0]
        target_tensor = training_pair[1]

        loss = train(input_tensor, target_tensor, encoder, decoder,
                     encoder_optimizer, decoder_optimizer, criterion)
        print_loss_total += loss
        plot_loss_total += loss

        if iter % print_every == 0:
            print_loss_avg = print_loss_total / print_every
            print_loss_total = 0
            print('%s (%d %d%%) %.4f' %
                  (timeSince(start, iter / n_iters), iter,
                   iter / n_iters * 100, print_loss_avg))

        if iter % plot_every == 0:
            plot_loss_avg = plot_loss_total / plot_every
            plot_losses.append(plot_loss_avg)
            plot_loss_total = 0

    showPlot(plot_losses)


# Plotting losses
import matplotlib.pyplot as plt
# plt.switch_backend('agg')
import matplotlib.ticker as ticker
import numpy as np


def showPlot(points):
    plt.figure()
    fig, ax = plt.subplots()
    # this locator puts ticks at regular intervals
    loc = ticker.MultipleLocator(base=0.2)
    ax.yaxis.set_major_locator(loc)
    plt.plot(points)


# Evaluation is mostly the same as training, but there are no targets so
# we simply feed the decoder's predictions back to itself for each step.
# Every time it predicts a word we add it to the output string, and if it
# predicts the EOS token we stop there.
def evaluate(encoder, decoder, sentence, max_length=MAX_LENGTH):
    with torch.no_grad():
        input_tensor = tensorFromSentence(input_lang, sentence)
        input_length = input_tensor.size()[0]
        encoder_hidden = encoder.initHidden()

        encoder_outputs = torch.zeros(max_length,
                                      encoder.hidden_size,
                                      device=device)

        for ei in range(input_length):
            encoder_output, encoder_hidden = encoder(input_tensor[ei],
                                                     encoder_hidden)
            encoder_outputs[ei] += encoder_output[0, 0]

        decoder_input = torch.tensor([[SOS_token]], device=device)  # SOS

        decoder_hidden = encoder_hidden

        decoded_words = []
        decoder_attentions = torch.zeros(max_length, max_length)

        for di in range(max_length):
            decoder_output, decoder_hidden, decoder_attention = decoder(
                decoder_input, decoder_hidden, encoder_outputs)
            decoder_attentions[di] = decoder_attention.data
            topv, topi = decoder_output.data.topk(1)
            if topi.item() == EOS_token:
                decoded_words.append('<EOS>')
                break
            else:
                decoded_words.append(output_lang.index2word[topi.item()])

            decoder_input = topi.squeeze().detach()

        return decoded_words, decoder_attentions[:di + 1]


# evaluate random sentences from the training set
def evaluateRandomly(encoder, decoder, n=10):
    print(n, 'examples of evaluation:')
    for i in range(n):
        pair = random.choice(pairs)
        print('>', pair[0])
        print('=', pair[1])
        output_words, attentions = evaluate(encoder, decoder, pair[0])
        output_sentence = ' '.join(output_words)
        print('<', output_sentence)
        print('')
    print('end')    


# Training and Evaluating
hidden_size = 256
encoder1 = EncoderRNN(input_lang.n_words, hidden_size).to(device)
attn_decoder1 = AttnDecoderRNN(hidden_size, output_lang.n_words,
                               dropout_p=0.1).to(device)

trainIters(encoder1, attn_decoder1, 50000, print_every=1000)

evaluateRandomly(encoder1, attn_decoder1)


# Visualizing Attention
# You could simply run ``plt.matshow(attentions)`` to see attention output
# displayed as a matrix, with the columns being input steps and rows being
# output steps:
output_words, attentions = evaluate(encoder1, attn_decoder1,
                                    "je suis trop froid .")
plt.matshow(attentions.numpy())
plt.show()


# For a better viewing experience we will do the extra work of adding axes
# and labels:
def showAttention(input_sentence, output_words, attentions):
    # Set up figure with colorbar
    fig = plt.figure()
    ax = fig.add_subplot(111)
    cax = ax.matshow(attentions.numpy(), cmap='bone')
    fig.colorbar(cax)

    # Set up axes
    ax.set_xticklabels([''] + input_sentence.split(' ') + ['<EOS>'],
                       rotation=90)
    ax.set_yticklabels([''] + output_words)

    # Show label at every tick
    ax.xaxis.set_major_locator(ticker.MultipleLocator(1))
    ax.yaxis.set_major_locator(ticker.MultipleLocator(1))

    plt.show()


def evaluateAndShowAttention(input_sentence):
    output_words, attentions = evaluate(encoder1, attn_decoder1,
                                        input_sentence)
    print('input =', input_sentence)
    print('output =', ' '.join(output_words))
    showAttention(input_sentence, output_words, attentions)


evaluateAndShowAttention("elle a cinq ans de moins que moi .")

evaluateAndShowAttention("elle est trop petit .")

evaluateAndShowAttention("je ne crains pas de mourir .")

evaluateAndShowAttention("c est un jeune directeur plein de talent .")

```





# 库函数

## torch

### arange

生成包含指定等差数列的一维张量。

```python
>>> torch.arange(5)
tensor([ 0,  1,  2,  3,  4])
>>> torch.arange(1, 4)
tensor([ 1,  2,  3])
>>> torch.arange(1, 2.5, 0.5)
tensor([ 1.0000,  1.5000,  2.0000])
```



### bmm

批量矩阵乘法。

```python
>>> input = torch.randn(10, 3, 4)
>>> mat2 = torch.randn(10, 4, 5)
>>> res = torch.bmm(input, mat2)
>>> res.size()
torch.Size([10, 3, 5])
```



### cat

拼接张量。

```python
>>> x = torch.randn(2, 3)
>>> x
tensor([[ 0.6580, -1.0969, -0.4614],
        [-0.1034, -0.5790,  0.1497]])
>>> torch.cat((x, x, x), 0)
tensor([[ 0.6580, -1.0969, -0.4614],
        [-0.1034, -0.5790,  0.1497],
        [ 0.6580, -1.0969, -0.4614],
        [-0.1034, -0.5790,  0.1497],
        [ 0.6580, -1.0969, -0.4614],
        [-0.1034, -0.5790,  0.1497]])
>>> torch.cat((x, x, x), 1)
tensor([[ 0.6580, -1.0969, -0.4614,  0.6580, -1.0969, -0.4614,  0.6580,
         -1.0969, -0.4614],
        [-0.1034, -0.5790,  0.1497, -0.1034, -0.5790,  0.1497, -0.1034,
         -0.5790,  0.1497]])
```



### clamp

将输入张量中的所有数应用下限和上限。

```python
>>> a = torch.randn(4)
>>> a
tensor([-1.7120,  0.1734, -0.0478, -0.0922])
>>> torch.clamp(a, min=-0.5, max=0.5)
tensor([-0.5000,  0.1734, -0.0478, -0.0922])

>>> a = torch.randn(4)
>>> a
tensor([-0.0299, -2.3184,  2.1593, -0.8883])
>>> torch.clamp(a, min=0.5)
tensor([ 0.5000,  0.5000,  2.1593,  0.5000])
```

`torch.clamp(x, min=0)`即为 ReLU 激活函数。



### equal

判断两个张量是否相等。

```python
>>> one1 = torch.ones(2,3)
>>> one2 = torch.ones(2,3)
>>> one1 == one2
tensor([[True, True, True],
        [True, True, True]])
>>> one1.equal(one2)
True
```



### flatten

将张量展开为向量。

```python
>>> t = torch.tensor([[[1, 2],
                       [3, 4]],
                      [[5, 6],
                       [7, 8]]])
>>> torch.flatten(t)
tensor([1, 2, 3, 4, 5, 6, 7, 8])
>>> torch.flatten(t, start_dim=1)
tensor([[1, 2, 3, 4],
        [5, 6, 7, 8]])
```



### mm

矩阵乘法。

```python
>>> mat1 = torch.randn(1, 3)
>>> mat2 = torch.randn(3, 1)
>>> torch.mm(mat1, mat2)
tensor([[0.0717]])
```



### ones

生成指定形状的全1张量。

```python
>>> torch.ones(2, 3)
tensor([[ 1.,  1.,  1.],
        [ 1.,  1.,  1.]])
```



### randn

生成指定形状的随机张量，其中每个元素服从标准正态分布。

```python
>>> torch.randn(2, 3)
tensor([[ 1.5954,  2.8929, -1.0923],
        [ 1.1719, -0.4709, -0.1996]])
```



### squeeze

返回一个张量，其在输入张量的基础上删除所有规模为1的维度。返回张量与输入张量共享内存。

```python
>>> input = torch.randn(1,2,1,3,1,4)
>>> input.shape
torch.Size([1, 2, 1, 3, 1, 4])
>>> output = input.unsqueeze()
>>> output.shape
torch.Size([2, 3, 4])
```





### Tensor

| Data type                                                    | dtype                                 | CPU tensor             | GPU tensor                  |
| ------------------------------------------------------------ | ------------------------------------- | ---------------------- | --------------------------- |
| 32-bit floating point                                        | `torch.float32` or `torch.float`      | `torch.FloatTensor`    | `torch.cuda.FloatTensor`    |
| 64-bit floating point                                        | `torch.float64` or `torch.double`     | `torch.DoubleTensor`   | `torch.cuda.DoubleTensor`   |
| 16-bit floating point [1](https://pytorch.org/docs/stable/tensors.html#id3) | `torch.float16` or `torch.half`       | `torch.HalfTensor`     | `torch.cuda.HalfTensor`     |
| 16-bit floating point [2](https://pytorch.org/docs/stable/tensors.html#id4) | `torch.bfloat16`                      | `torch.BFloat16Tensor` | `torch.cuda.BFloat16Tensor` |
| 32-bit complex                                               | `torch.complex32`                     |                        |                             |
| 64-bit complex                                               | `torch.complex64`                     |                        |                             |
| 128-bit complex                                              | `torch.complex128` or `torch.cdouble` |                        |                             |
| 8-bit integer (unsigned)                                     | `torch.uint8`                         | `torch.ByteTensor`     | `torch.cuda.ByteTensor`     |
| 8-bit integer (signed)                                       | `torch.int8`                          | `torch.CharTensor`     | `torch.cuda.CharTensor`     |
| 16-bit integer (signed)                                      | `torch.int16` or `torch.short`        | `torch.ShortTensor`    | `torch.cuda.ShortTensor`    |
| 32-bit integer (signed)                                      | `torch.int32` or `torch.int`          | `torch.IntTensor`      | `torch.cuda.IntTensor`      |
| 64-bit integer (signed)                                      | `torch.int64` or `torch.long`         | `torch.LongTensor`     | `torch.cuda.LongTensor`     |
| Boolean                                                      | `torch.bool`                          | `torch.BoolTensor`     | `torch.cuda.BoolTensor`     |

```python
a = np.array([1.,2,3])
t64 = torch.tensor(a)                       # t64.dtype = torch.float64
t32 = torch.tensor(a, dtype=torch.float32)  # t32.dtype = torch.float32
t32 = t64.float()                           # 类型转换
                                            # 注意nn中各层接受的tensor类型一般为float32
```



**detach**

返回一个张量，其与输入张量共享内存，但在计算图之外，不参与梯度计算。

```python
# 1
>>> a = torch.tensor([1, 2, 3.], requires_grad=True)
>>> out = a.sigmoid()
>>> out.sum().backward()
>>> a.grad
tensor([1., 1., 1.])

# 2
>>> a = torch.tensor([1, 2, 3.], requires_grad=True)
>>> out = a.sigmoid()
>>> c = out.detach()
>>> out.sum().backward()  # 可以计算梯度 
>>> a.grad
tensor([0.1966, 0.1050, 0.0452])

# 3
>>> a = torch.tensor([1, 2, 3.], requires_grad=True)
>>> out = a.sigmoid()
>>> c = out.detach()
>>> c.sum().backward()    # c不能计算梯度
RuntimeError: element 0 of tensors does not require grad and does not have a grad_fn

# 4
>>> a = torch.tensor([1, 2, 3.], requires_grad=True)
>>> out = a.sigmoid()
>>> c = out.detach()
>>> c.zero_()
>>> out.sum().backward()    # out的值被修改而不能计算梯度
RuntimeError: one of the variables needed for gradient computation has been modified by an inplace operation: ……
```



**item**

对于只有一个元素的张量，返回该元素的值。

```python
>>> t = torch.tensor([[[1]]])
>>> t.shape
torch.Size([1, 1, 1])
>>> t.item()
1
```





### topk

返回一维张量的最大的k个数。对于二维张量，返回每行的最大的k个数。

```python
>>> a = torch.arange(6)
>>> a.topk(1)
torch.return_types.topk(values=tensor([5]),indices=tensor([5]))
>>> a.topk(3)
torch.return_types.topk(values=tensor([5, 4, 3]),indices=tensor([5, 4, 3]))

>>> a = torch.arange(6).view(2,3)
>>> v, i = a.topk(1)
>>> v
tensor([[2],
        [5]]) 
>>> i
tensor([[2],
        [2]])

```





### unsqueeze

返回一个张量，其在输入张量的基础上在指定位置增加一个规模为1的维度。返回张量与输入张量共享内存。

```python
>>> input = torch.randn(2,3,4)
>>> input.shape
torch.Size([2, 3, 4])
>>> output = input.unsqueeze(0)
>>> output.shape
torch.Size([1, 2, 3, 4])
>>> output = input.unsqueeze(3)
>>> output.shape
torch.Size([2, 3, 4, 1])
```



### zeros

生成指定形状的全0张量。

```python
>>> torch.zeros(2, 3)
tensor([[ 0.,  0.,  0.],
        [ 0.,  0.,  0.]])
```





## torch.nn



### Conv1d

一维卷积层。

```python
>>> m1 = nn.Conv1d(1, 32, 3, 1)                 # 卷积核长度为3,步长为1
>>> m2 = nn.Conv1d(1, 32, 3, 3)                 # 步长为3
>>> m3 = nn.Conv1d(1, 32, 3, 3, padding=(1,1))  # 左右各用1个零填充
>>> input = torch.rand((100, 1, 28))
>>> output1, output2, output3= m1(input), m2(input), m3(input)
>>> output1.shape
torch.Size([100, 32, 26])
>>> output2.shape
torch.Size([100, 32, 9])
>>> output3.shape
torch.Size([100, 32, 10])
```



### Conv2d

二维卷积层。

```python
>>> m1 = nn.Conv2d(1, 32, 3, 1)                 # 卷积核大小为(3,3),步长为1
												# 将1个通道映射到32个卷积特征/通道
>>> m2 = nn.Conv2d(1, 32, (3,5), 1)             # 卷积核大小为(3,5)
>>> m3 = nn.Conv2d(1, 32, 3, 3)                 # 步长为3
>>> m4 = nn.Conv2d(1, 32, 3, 3, padding=(1,1))  # 上下,左右各用1,1行零填充
>>> input = torch.rand((100, 1, 28, 28))
>>> output1, output2, output3, output4= m1(input), m2(input), m3(input), m4(input)
>>> output1.shape
torch.Size([100, 32, 26, 26])
>>> output2.shape
torch.Size([100, 32, 26, 24])
>>> output3.shape
torch.Size([100, 32, 9, 9])
>>> output4.shape
torch.Size([100, 32, 10, 10])
```



### CrossEntropyLoss

交叉熵损失函数。见`torch.nn.NLLLoss`。

```python
>>> loss = nn.CrossEntropyLoss()
>>> a1 = torch.tensor([[0.1, 0.8, 0.1]])
>>> a2 = torch.tensor([1])
>>> b = loss(a1, a2)
>>> b
tensor(0.6897)
>>> a2 = torch.tensor([0])
>>> b = loss(a1, a2)
>>> b
tensor(1.3897)

# CrossEntropyLoss() = softmax() + log() + NLLLoss()
>>> loss = nn.CrossEntropyLoss()
>>> input = torch.tensor([[ 0.4377, -0.3976, -1.3221],
                          [ 1.8402, -0.1696,  0.4744],
                          [-3.4641, -0.2303,  0.3552]])
>>> target = torch.tensor([0, 1, 2])
>>> loss(input, target)
tensor(1.0896)
```



### Dropout

以给定概率将张量中的每个数置零，剩余的数乘以$$1/(1-p)$$。每次使用Dropout层的结果是随机的。

```python
>>> m = nn.Dropout(0.5)
>>> input = torch.randn(4, 4)
>>> output = m(input)
>>> input
tensor([[-1.1218,  0.1338, -0.0065, -1.6416],
        [ 0.8897, -1.6002, -0.6922,  0.0689],
        [-1.3392, -0.5207, -0.2739, -0.9653],
        [ 0.6608,  0.9212,  0.0579,  0.9670]])
>>> output
tensor([[-0.0000,  0.2677, -0.0000, -3.2831],
        [ 1.7795, -3.2004, -1.3843,  0.0000],
        [-0.0000, -0.0000, -0.0000, -0.0000],
        [ 0.0000,  1.8425,  0.1158,  0.0000]])
```



### Dropout2d

以给定概率将张量$$(N,C,H,W)$$的每个通道置零，剩余的通道乘以$$1/(1-p)$$。每次使用Dropout层的结果是随机的。

```python
>>> m = nn.Dropout2d(0.5)
>>> input = torch.randn(1, 8, 2, 2)
>>> output = m(input)
>>> input
tensor([[[[ 1.7200, -0.7948],
          [-0.1551, -0.8467]],

         [[-1.0479, -0.6172],
          [-0.8419, -0.8668]],

         [[ 0.4776,  1.7682],
          [ 1.0376,  0.8871]],

         [[-0.8826,  1.5624],
          [ 1.4573, -0.0573]],

         [[-1.4288, -0.6288],
          [ 1.2000,  1.3250]],

         [[ 1.8099,  0.7262],
          [-0.5595,  1.4562]],

         [[ 0.7452, -2.1875],
          [ 0.0116,  0.5224]],

         [[ 0.1152,  0.1012],
          [ 0.5634, -0.1202]]]])
>>> output
tensor([[[[ 0.0000, -0.0000],
          [-0.0000, -0.0000]],

         [[-2.0957, -1.2344],
          [-1.6837, -1.7336]],

         [[ 0.9551,  3.5364],
          [ 2.0752,  1.7741]],

         [[-1.7651,  3.1247],
          [ 2.9146, -0.1147]],

         [[-0.0000, -0.0000],
          [ 0.0000,  0.0000]],

         [[ 3.6198,  1.4524],
          [-1.1190,  2.9124]],

         [[ 0.0000, -0.0000],
          [ 0.0000,  0.0000]],

         [[ 0.0000,  0.0000],
          [ 0.0000, -0.0000]]]])
```



### Embedding

嵌入层。

```python
>>> embedding = nn.Embedding(10, 3)  # 词汇表规模 = 10, 嵌入维数 = 3, 共30个参数
>>> input = torch.LongTensor([[1,2,4,5],[4,3,2,9]])
>>> embedding(input)
tensor([[[-0.0251, -1.6902,  0.7172],
         [-0.6431,  0.0748,  0.6969],
         [ 1.4970,  1.3448, -0.9685],
         [-0.3677, -2.7265, -0.1685]],

        [[ 1.4970,  1.3448, -0.9685],
         [ 0.4362, -0.4004,  0.9400],
         [-0.6431,  0.0748,  0.6969],
         [ 0.9124, -2.3616,  1.1151]]])
```



### GRU

GRU 层。

```python
>>> rnn = nn.GRU(5, 10, 2)           # GRU可以视作简化的LSTM,各参数含义与LSTM相同
>>> input = torch.randn(20, 64, 5)
>>> h0 = torch.randn(2, 64, 10)
>>> output, hn = rnn(input, h0)
>>> output.shape
torch.Size([20, 64, 10])
>>> hn.shape
torch.Size([2, 64, 10])
```



### L1Loss

平均绝对误差损失函数。

```python
>>> a1 = torch.arange(10.0)
>>> a2 = a1+2
>>> loss = nn.L1Loss()
>>> b = loss(a1, a2)
>>> b
tensor(2.)
>>> loss = nn.MSELoss(reduction='sum')
>>> b = loss(a1, a2)
>>> b
tensor(20.)
```



### Linear

全连接层。

```python
>>> m = nn.Linear(20, 4)
>>> input = torch.randn(128, 20)
>>> output = m(input)
>>> print(output.size())
torch.Size([128, 4])
```



### LSTM

> 参考：[理解Pytorch中LSTM的输入输出参数含义](https://www.cnblogs.com/marsggbo/p/12123755.html)

LSTM 层。

```python
>>> rnn = nn.LSTM(5, 10, 2, [dropout=0.5]) # 输入向量x的维数为5,隐状态h的维数为10,堆叠2层
										   # 在每层(最上层除外)的输出位置增加一个dropout层
									       # 多层LSTM中,上层的输入是下层的隐状态
>>> input = torch.randn(20, 64, 5)         # 一批64个序列,每个序列有20个5维向量
>>> h0 = torch.randn(2, 64, 10)            # 第一个参数为层数与方向数的乘积,单向和双向LSTM
										   #   的方向数分别为1和2
    									   # 第二个参数为输入序列的数量
>>> c0 = torch.randn(2, 64, 10)            # 第三个参数为隐状态维数
>>> output, (hn, cn) = rnn(input, (h0, c0)) # 输入h,c的初值,输出h,c的终值
											# 若不输入初值,则默认为0
>>> output.shape
torch.Size([20, 64, 10])                   # 从前往后输出最上层的所有隐状态
>>> hn.shape
torch.Size([2, 64, 10])                    # 输出每一(层,方向)的最终隐状态
                                           # 对于单向LSTM, hn[-1]==output[-1]


>>> rnn = nn.LSTM(5, 10, 2, bidirectional=True)  # 双向LSTM,相当于将输入向量正向和反向各
                                                 #   输入一次
>>> input = torch.randn(20, 64, 5)
>>> h0 = torch.randn(4, 64, 10)                  # 层数*方向数=4
>>> c0 = torch.randn(4, 64, 10)
>>> output, (hn, cn) = rnn(input, (h0, c0))
>>> output.shape
torch.Size([20, 64, 20])                   # 输出最上层的所有隐状态,拼接正向与反向的输出
>>> hn.shape
torch.Size([4, 64, 10])                    # 每一(层,方向)的最终隐状态
```



### MaxPool1d

一维最大汇聚层。见torch.nn.functional.max_pool1d。

```python
>>> m1 = nn.MaxPool1d(2, stride=1)
>>> m2 = nn.MaxPool1d(2, stride=2)
>>> input = torch.randn(1, 8)
>>> output1 = m1(input)
>>> output2 = m2(input)
>>> input
tensor([[ 0.3055,  0.5521,  1.9417, -0.7325,  0.3202, -1.4555,  1.7270,  3.1311]])
>>> output1
tensor([[0.5521, 1.9417, 1.9417, 0.3202, 0.3202, 1.7270, 3.1311]])
>>> output2
tensor([[0.5521, 1.9417, 0.3202, 3.1311]])
```



### MaxPool2d

二维最大汇聚层。见torch.nn.functional.max_pool2d。

```python
>>> m1 = nn.MaxPool2d(2, stride=1)
>>> m2 = nn.MaxPool2d(2, stride=2)
>>> input = torch.randn(1, 1, 4, 4)
>>> output1 = m1(input)
>>> output2 = m2(input)
>>> input
tensor([[[[-0.5308,  1.2014, -1.3582,  1.1337],
          [ 0.2359,  0.9501,  1.1915,  0.3432],
          [-1.4260, -0.1276, -2.2615,  0.8555],
          [-0.8545,  0.5436,  1.6482,  1.2749]]]])
>>> output1
tensor([[[[1.2014, 1.2014, 1.1915],
          [0.9501, 1.1915, 1.1915],
          [0.5436, 1.6482, 1.6482]]]])
>>> output2
tensor([[[[1.2014, 1.1915],
          [0.5436, 1.6482]]]])
```



### MSELoss

均方差损失函数。

```python
>>> a1 = torch.arange(10.0)
>>> a2 = a1+2
>>> loss = nn.MSELoss()
>>> b = loss(a1, a2)
>>> b
tensor(4.)
>>> loss = nn.MSELoss(reduction='sum')
>>> b = loss(a1, a2)
>>> b
tensor(40.)
```



### NLLLoss

见`torch.nn.CrossEntropyLoss`。

```python
>>> loss = nn.NLLLoss()
>>> input = torch.tensor([[ 0.4377, -0.3976, -1.3221],
                          [ 1.8402, -0.1696,  0.4744],
                          [-3.4641, -0.2303,  0.3552]])
>>> input = input.softmax(dim=1)
>>> input = input.log()
>>> target = torch.tensor([0, 1, 2])
>>> loss(input, target)
tensor(1.0896)

```



### ReLU

ReLU 激活函数层。见torch.nn.functional.relu。

```python
>>> m = nn.ReLU()
>>> input = torch.randn(2)
>>> output = m(input)
>>> input
tensor([ 1.2175, -0.7772,  1.3282, -0.1987,  0.3403, -1.3309, -0.3600, -1.5150])
>>> output
tensor([1.2175, 0.0000, 1.3282, 0.0000, 0.3403, 0.0000, 0.0000, 0.0000])
```



### Softmax

Softmax层。torch.nn.LogSoftmax相当于在此基础上为每个输出值求（自然）对数。

```python
>>> m1 = nn.Softmax(dim=0)
>>> m2 = nn.LogSoftmax(dim=0)
>>> input = torch.arange(4.0)
>>> output1 = m1(input)
>>> output2 = m2(input)
>>> input
tensor([0., 1., 2., 3.])
>>> output1
tensor([0.0321, 0.0871, 0.2369, 0.6439])
>>> output2
tensor([-3.4402, -2.4402, -1.4402, -0.4402])
```



## torch.nn.functional



### max_pool1d

一维最大汇聚函数。见torch.nn.MaxPool1d。

```python
>>> input = torch.randn(1, 8)
>>> output1 = F.max_pool1d(input, 2, 1)
>>> output2 = F.max_pool1d(input, 2)
>>> input
tensor([[ 0.3055,  0.5521,  1.9417, -0.7325,  0.3202, -1.4555,  1.7270,  3.1311]])
>>> output1
tensor([[0.5521, 1.9417, 1.9417, 0.3202, 0.3202, 1.7270, 3.1311]])
>>> output2
tensor([[0.5521, 1.9417, 0.3202, 3.1311]])
```



### max_pool2d

二维最大汇聚函数。见torch.nn.MaxPool2d。

```python
>>> input = torch.randn(1, 1, 4, 4)
>>> output1 = F.max_pool2d(input, 2, 1)
>>> output2 = F.max_pool2d(input, 2)
>>> input
tensor([[[[-0.5308,  1.2014, -1.3582,  1.1337],
          [ 0.2359,  0.9501,  1.1915,  0.3432],
          [-1.4260, -0.1276, -2.2615,  0.8555],
          [-0.8545,  0.5436,  1.6482,  1.2749]]]])
>>> output1
tensor([[[[1.2014, 1.2014, 1.1915],
          [0.9501, 1.1915, 1.1915],
          [0.5436, 1.6482, 1.6482]]]])
>>> output2
tensor([[[[1.2014, 1.1915],
          [0.5436, 1.6482]]]])
```



### one_hot

将向量转换为one-hot表示。

```python
>>> F.one_hot(torch.arange(0, 5))
tensor([[1, 0, 0, 0, 0],
        [0, 1, 0, 0, 0],
        [0, 0, 1, 0, 0],
        [0, 0, 0, 1, 0],
        [0, 0, 0, 0, 1]])

>>> F.one_hot(torch.arange(0, 5) % 3, num_classes=5)
tensor([[1, 0, 0, 0, 0],
        [0, 1, 0, 0, 0],
        [0, 0, 1, 0, 0],
        [1, 0, 0, 0, 0],
        [0, 1, 0, 0, 0]])
```



### relu

ReLU 激活函数。见torch.nn.ReLU。

```python
>>> input = torch.randn(8)
>>> output = F.relu(input)
>>> input
tensor([ 1.2175, -0.7772,  1.3282, -0.1987,  0.3403, -1.3309, -0.3600, -1.5150])
>>> output
tensor([1.2175, 0.0000, 1.3282, 0.0000, 0.3403, 0.0000, 0.0000, 0.0000])
```



### softmax

softmax回归。

```python
>>> input = torch.tensor([[0.1,0.2,0.3,1],[0.2,0.3,0.4,2]])
>>> output0 = F.softmax(input, dim=0) # for every column
>>> output1 = F.softmax(input, dim=1) # for every row
>>> input
tensor([[0.1000, 0.2000, 0.3000, 1.0000],
        [0.2000, 0.3000, 0.4000, 2.0000]])
>>> output0
tensor([[0.4750, 0.4750, 0.4750, 0.2689],
        [0.5250, 0.5250, 0.5250, 0.7311]])
>>> output1
tensor([[0.1728, 0.1910, 0.2111, 0.4251],
        [0.1067, 0.1179, 0.1303, 0.6452]])
```



### tanh

tanh 激活函数。

```python

```





## torch.view

view相当于numpy中的resize功能，即改变张量的形状。

```python
>>> a = torch.arange(12)
>>> b = a.view(2, 6)
>>> c = b.view(1, -1) # -1位置的参数会根据元素总数和其它维度的长度计算
>>> a
tensor([ 0,  1,  2,  3,  4,  5,  6,  7,  8,  9, 10, 11])
>>> b
tensor([[ 0,  1,  2,  3,  4,  5],
        [ 6,  7,  8,  9, 10, 11]])
>>> c
tensor([[ 0,  1,  2,  3,  4,  5,  6,  7,  8,  9, 10, 11]])
```



## torch.optim

```python
optimizer = optim.SGD(model.parameters(), lr=0.01)
                 #梯度下降法  需要学习的参数  学习率
```



## torch.utils.data




