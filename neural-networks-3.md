---
layout: page
permalink: /neural-networks-3/
---

Table of Contents:

- [그라디언트 점검 (Gradient checks)](#gradcheck)
- [Sanity checks](#sanitycheck)
- [학습 과정 돌보기 (Babysitting the learning process)](#baby)
  - [손실 함수 (Loss function)](#loss)
  - [훈련/검증 성능 (Train/val accuracy)](#accuracy)
  - [웨이트의 현재값과 변화량의 비율 (Weights:Updates ratio)](#ratio)
  - [레이어별 활성값 및 그라디언트값의 분포 (Activation/Gradient distributions per layer)](#distr)
  - [시각화 (Visualization)](#vis)
- [파라미터 업데이트 (Parameter updates)](#update)
  - [일차 근사 방법 (SGD) (First-order (SGD)), 모멘텀 (momentum), Nesterov 모멘텀 (Nesterov momentum)](#sgd)
  - [학습 속도를 담금질하기 (Annealing the learning rate)](#anneal)
  - [이차 근사 방법 (Second-order methods)](#second)
  - [파라미터별로 학습 속도를 데이터가 판단하게 하기 (Adagrad, RMSProp) )Per-parameter adaptive learning rates (Adagrad, RMSProp))](#ada)
- [초-파라미터 최적화 (Hyperparameter Optimization)](#hyper)
- [평가 (Evaluation)](#eval)
  - [모형 앙상블 (Model Ensembles)](#ensemble)
- [요약](#summary)
- [추가적인 참고 문헌](#add)

## Learning

이전 섹션들에서는 레이어를 몇 층 쌓고 레이어별로 몇 개의 유닛을 준비할지(newwork connectivity), 데이터를 어떻게 준비하고 어떤 손실 함수(loss function)를 선택할지 논하였다. 말하자면 이전 섹션들은 주로 뉴럴 네트워크(Neural Network)의 정적인 부분인데, 본 섹션에서는 동적인 부분들을 소개한다. 파라미터(parameter)를 학습하고 좋은 초-파라미터(hyperparamter)를 찾는 과정 등을 다룰 예정이다.

<a name='gradcheck'></a>
### 그라디언트 체크 (Gradient Checks)

이론적인 그라디언트 체크라 하면, 수치적으로 계산한(numerical) 그라디언트와 수식으로 계산한(analytic) 그라디언트를 비교하는 정도라 매우 간단하다고 생각할 수도 있겠다. 그렇지만 이 작업을 직접 실현해 보면 훨씬 복잡하고 뜬금없이 오차가 발생하기도 쉽다는 것을 깨달을 것이다. 이제 팁, 트릭, 조심할 이슈들 몇 개를 소개하고자 한다.


**같은 근사라 하여도 이론적으로 더 정확도가 높은 근사 공식이 있다 (Use the centered formula)**. 그라디언트($\frac{df(x)}{dx}$)를 수치적으로 근사한다 하면 보통 다음 유한 차분 근사(finite difference approximation)를 떠올릴 것이다:

$$
\frac{df(x)}{dx} = \frac{f(x + h) - f(x)}{h} \hspace{0.1in} \text{(bad, do not use)}
$$

여기서 $h$는 아주 작은 수이고 보통 1e-5 정도의 수를 사용한다. 위 식보다는 아래의 *중심화된(centered)* 차분 공식이 경험적으로는 훨씬 낫다:

$$
\frac{df(x)}{dx} = \frac{f(x + h) - f(x - h)}{2h} \hspace{0.1in} \text{(use instead)}
$$

물론 이 공식은 $f(x+h)$ 말고도 $f(x-h)$도 계산하여야 하므로 최초 식보다 계산량이 두 배 많지만 훨씬 정확한 근사를 제공한다. $f(x+h)$ 및 $f(x-h)$의 ($x$ 근방에서의) 테일러 전개를 고려하면 이유를 금방 알 수 있다. 첫 식은 $O(h)$의 오차가 있는 데 반해 두번째 식은 오차가 $O(h^2)$이다 (즉, 이차 근사이다).  -- 역자 주 : (1) 테일러 전개에서 $f(x + h) = f(x) + hf'(x) + O(h)$로부터 $f'(x) - \frac{(f(x+h)-f(x)}{h} = O(h)$. (2) $h$가 보통 벡터이므로 $O(h)$보다는 $O(\|h\|)$가 더 정확한 표현이나 편의상 $\|\cdot\|$을 생략한 듯 보입니다.


**상대 오차를 사용하라 (Use relative error for the comparison)**. 그라디언트의 (수식으로 계산한, analytic) 참값 $f'_a$와 수치적(numerical) 근사값 $f'_n$을 비교하려면 어떤 디테일을 점검하여야 할까? 이 둘이 비슷하지 않음(not compatible)을 어떻게 알아낼 수 있을까? 가장 쉽게는 둘의 절대 오차 $\mid f'_a - f'_n \mid $ 혹은 그 제곱을 쭉 추적하여 이 값(들)이 언젠가 어느 한계점(threshold)를 넘으면 그라디언트 오류라 할 수도 있겠다. 그렇지만 절대 오차에는 문제가 있는 것이, 가령 절대 오차가 1e-4라 가정하여 보자. 만약 $f'_a$와 $f'_n$ 모두 1.0 언저리라면 1e-4의 오차 정도는 매우 훌륭한 근사이고  $f'_a \approx f'_n$이라 할 수 있다. 그런데 만약 두 그라디언트가 1e-5거나 더 작은 값이라면? 그렇다면 1e-4는 매우 큰 차이가 되고 근사가 실패했다고 보아야 한다. 따라서 절대 오차와 두 그라디언트 값의 비율을 고려하는 *상대 오차*가 더 적절하다. 언제나!: 


$$
\frac{\mid f'_a - f'_n \mid}{\max(\mid f'_a \mid, \mid f'_n \mid)}
$$

보통의 상대 오차 공식은 분모에 $f'_a$ 혹은 $f'_n$ 둘 중 하나만 있지만, 나는 둘의 최대값을 분모로 선호하는 편이다. 그래야 공식에 대칭성이 생기고 둘 중 하나가 exactly 0이 되어 분모가 0이 되는 사태를 방지할 수 있다 (ReLU를 사용하면 자주 일어나는 문제이다). $f'_a$와 $f'_n$가 모두 exact 0이 된다면? 이 때는 상대 오차를 점검할 필요 없이 그라디언트 체크를 통과하여야 한다. 당신의 코드가 이 상황을 감안하여 조직된 코드인지 점검하여 보라.  

실제 상황에서의 유용한 가이드:

- (상대 오차) > 1e-2 면 그라디언트 계산이 아마 잘못되었을 수도 있다.
- 1e-2 > (상대 오차) > 1e-4 면 불편함을 느끼기 바란다.
- 1e-4 > (상대 오차) 는, 꺾임이 있는 목적함수 (objectives with kinks)에서는 괜찮다. 그렇지만 tanh 혹은 softmax를 쓰는 목적함수처럼 꺾임이 없다면 1e-4는 너무 크다.
- 1e-7 혹은 그보다 작은 상대 오차라면, 행복을 느껴야 한다.

하나 더 유념해야 할 것은, 망의 레이어 개수가 많아지면(deeper network) 상대 오차가 커진다. 이를테면 레이어(layer) 10개짜리 망(network)에서 인풋 데이터의 그라디언트를 체크한다면, 에러가 층을 올라가며 축적되므로 1e-2 정도의 상대 오차는 괜찮을 수도 있다. 거꾸로 말하자면, 미분가능한 함수 하나만 갖고 노는데 1e-2의 상대 오차가 발생한다면 이것은 부정확한 그라디언트일 가능성이 매우 높다.


**이중정확성 변수를 사용하라 (Use double precision)**. 흔히들 실수하는 것이, 그라디언트 체크를 계산하는 데 단일정확성 부동소숫점(single precision floating point) 변수를 사용하는 경우가 있다. 단일정확성 변수를 쓰면 그라디언트 계산이 맞다 하더라도 상대 오차가 (1e-2 정도로) 커지는 경우가 종종 있다. 내 경험상으로는 이중정확성 변수를 쓰면 상대 오차가 1e-2에서 1e-8까지 개선되는 경우도 봤다.  


**부동소숫점 연산이 활성화되는 범위에서 계산하라 (Stick around active range of floating point)**. 당신 좀더 세심한 코드를 작성하고 실수를 줄이려면 ["모든 컴퓨터 사이언티스트들이 부동소숫점 연산에 대해 알아야 하는 것들(What Every Computer Scientist Should Know About Floating-Point Arithmetic)"](http://docs.oracle.com/cd/E19957-01/806-3568/ncg_goldberg.html) 를 읽는 게 좋다. 예를 들어, 신경망에서는 손실함수(loss function)를 배치별로(over batch)로 normalize하는 것이 보통이다 (역자 주 : 그라디언트 합을 배치 사이즈로 나누는 장면을 지칭하는 듯). 그렇지만 한 자료당(per datapoint) 그라디언트가 매우 작다면, 거기에 또 데이터 갯수를 *부가적으로* 나눌 경우 매우 작은 수가 되고 더욱더 많은 수치적인 문제가 생길 수 있다. 그래서 필자는 $f'_a$ 혹은 $f'_n$의 계산값을 계속 찍어보고 두 값이 너무 작지 않은가 확인하는 편이다. (대충 1e-10 혹은 그보다 작은 크기의 값이면 걱정하여라) 만약 두 값이 너무 작다면, 적당히 상수를 곱하여 부동소숫점 표현이 조금 더 "괜찮도록" (부동소숫점 표현에서 지수 부분이 0이 되도록) 만들 수도 있다. 


**목적함수에서의 꺾인 점 (Kinks in the objective)**. *꺾인 점(kink)*들에서 부정확한 계산이 발생할 수 있는데  이를  그라디언트 체크 과정에서도 염두에 두고 있어야 한다. 꺾인 점(kink)은 목적함수의 미분 불가능한 부분을 지칭하는 용어이다. ReLU 함수 ($max(0,x)$), 서포트 벡터 머신(SVM) 목적함수나 맥스아웃 뉴런(maxout neuron) 등을 사용하면 발생할 수 있다. 꺾인 점이 야기시킬 수 있는 문제는 대략 이렇다. ReLU 함수의 그라디언트를 $x = -1e6$에서 체크한다고 생각하여 보자. $x < 0$이므로 $f'_a$는 정확히 $0$이다. 그렇지만, 수치적으로 계산된 그라디언트는 $f(x+h)$가 꺾인 점을 넘을 수도 있으므로 (이를테면 $h > 1e-6$인 경우) 갑자기 $0$이 아닌 값을 내놓게 될 수도 있다. 이런 병적인(?) 경우까지 신경써야 하냐고 물을 수도 있겠는데, 사실 매우 흔하다. 예를 들어 CIFAR-10를 위해 서포트 벡터 머신(SVM)을 쓴다고 하면, 데이터가 50,000개이고(50,000 examples) 한 데이터당 $max(0,x)$ 항이 9개씩 있으니 결국 45만개의 ReLU항과 맞닥뜨리게 된다. 게다가 서포트 벡터 머신 분류기(SVM classifier)와 신경망(neural network)을 붙이면 ReLU들 때문에 꺾인 점이 더 늘어날 수도 있다. 

다행히도, 손실함수를 계산할 때 꺾인 점을 넘어서 계산했는지 (a kink was crossed) 여부를 알 수 있다. $max(x,y)$ 꼴 함수에서 $x$, $y$ 중 누가 "이겼는지"를 계속 기록해둔다고 생각해 보자.  $f(x+h)$와 $f(x-h)$를 계산할 때 적어도 하나의 "승자"가 바뀐다면, 꺾인 점을 넘는 현상이 발생한 것이고 그렇다면 수치적인 그라디언트가 정확한 값이 아닐 수도 있다.

**적은 수의 데이터만 써라 (Use only few datapoints)** 꺾인 점과 관련된 하나의 해결책은 더 적은 데이터를 쓰는 것이다. 손실함수가 꺾인 점을 포함하고 있으면 (ReLU나 margin loss등을 썼을 경우처럼) 데이터가 적을수록 더 적은 꺾인 점을 포함할 것이고, 따라서 유한 차분 근사(finite different approximation) 과정에서 꺾인 점을 가로지르는 경우가 더 적을 것이다. 게다가,   ~2 혹은 3개의 데이터에 대해서만 그라디언트 체크를 수행하는 게 거의 배치(batch) 전부에 대해 그라디언트 체크하는 게 될 테니 훨씬 빠르고 효율적이다. (역자 주 : 그렇지만 배치 사이즈가 작아지면 다른 쪽에서 문제가 생길 수도 있을 것 같은데..)



**Be careful with the step size h**. It is not necessarily the case that smaller is better, because when $h$ is much smaller, you may start running into numerical precision problems. Sometimes when the gradient doesn't check, it is possible that you change $h$ to be 1e-4 or 1e-6 and suddenly the gradient will be correct. This [wikipedia article](http://en.wikipedia.org/wiki/Numerical_differentiation) contains a chart that plots the value of **h** on the x-axis and the numerical gradient error on the y-axis.

**Gradcheck during a "characteristic" mode of operation**. It is important to realize that a gradient check is performed at a particular (and usually random), single point in the space of parameters. Even if the gradient check succeeds at that point, it is not immediately certain that the gradient is correctly implemented globally. Additionally, a random initialization might not be the most "characteristic" point in the space of parameters and may in fact introduce pathological situations where the gradient seems to be correctly implemented but isn't. For instance, an SVM with very small weight initialization will assign almost exactly zero scores to all datapoints and the gradients will exhibit a particular pattern across all datapoints. An incorrect implementation of the gradient could still produce this pattern and not generalize to a more characteristic mode of operation where some scores are larger than others. Therefore, to be safe it is best to use a short **burn-in** time during which the network is allowed to learn and perform the gradient check after the loss starts to go down. The danger of performing it at the first iteration is that this could introduce pathological edge cases and mask an incorrect implementation of the gradient.

**Don't let the regularization overwhelm the data**. It is often the case that a loss function is a sum of the data loss and the regularization loss (e.g. L2 penalty on weights). One danger to be aware of is that the regularization loss may overwhelm the data loss, in which case the gradients will be primarily coming from the regularization term (which usually has a much simpler gradient expression). This can mask an incorrect implementation of the data loss gradient. Therefore, it is recommended to turn off regularization and check the data loss alone first, and then the regularization term second and independently. One way to perform the latter is to hack the code to remove the data loss contribution. Another way is to increase the regularization strength so as to ensure that its effect is non-negligible in the gradient check, and that an incorrect implementation would be spotted.

**Remember to turn off dropout/augmentations**. When performing gradient check, remember to turn off any non-deterministic effects in the network, such as dropout, random data augmentations, etc. Otherwise these can clearly introduce huge errors when estimating the numerical gradient. The downside of turning off these effects is that you wouldn't be gradient checking them (e.g. it might be that dropout isn't backpropagated correctly). Therefore, a better solution might be to force a particular random seed before evaluating both $f(x+h)$ and $f(x-h)$, and when evaluating the analytic gradient.

**Check only few dimensions**. In practice the gradients can have sizes of million parameters. In these cases it is only practical to check some of the dimensions of the gradient and assume that the others are correct. **Be careful**: One issue to be careful with is to make sure to gradient check a few dimensions for every separate parameter. In some applications, people combine the parameters into a single large parameter vector for convenience. In these cases, for example, the biases could only take up a tiny number of parameters from the whole vector, so it is important to not sample at random but to take this into account and check that all parameters receive the correct gradients.

<a name='sanitycheck'></a>
### Before learning: sanity checks Tips/Tricks

Here are a few sanity checks you might consider running before you plunge into expensive optimization:

- **Look for correct loss at chance performance.** Make sure you're getting the loss you expect when you initialize with small parameters. It's best to first check the data loss alone (so set regularization strength to zero). For example, for CIFAR-10 with a Softmax classifier we would expect the initial loss to be 2.302, because we expect a diffuse probability of 0.1 for each class (since there are 10 classes), and Softmax loss is the negative log probability of the correct class so: -ln(0.1) = 2.302. For The Weston Watkins SVM, we expect all desired margins to be violated (since all scores are approximately zero), and hence expect a loss of 9 (since margin is 1 for each wrong class). If you're not seeing these losses there might be issue with initialization.
- As a second sanity check, increasing the regularization strength should increase the loss
- **Overfit a tiny subset of data**. Lastly and most importantly, before training on the full dataset try to train on a tiny portion (e.g. 20 examples) of your data and make sure you can achieve zero cost. For this experiment it's also best to set regularization to zero, otherwise this can prevent you from getting zero cost. Unless you pass this sanity check with a small dataset it is not worth proceeding to the full dataset. Note that it may happen that you can overfit very small dataset but still have an incorrect implementation. For instance, if your datapoints' features are random due to some bug, then it will be possible to overfit your small training set but you will never notice any generalization when you fold it your full dataset.

<a name='baby'></a>
### Babysitting the learning process

There are multiple useful quantities you should monitor during training of a neural network. These plots are the window into the training process and should be utilized to get intuitions about different hyperparameter settings and how they should be changed for more efficient learning. 

The x-axis of the plots below are always in units of epochs, which measure how many times every example has been seen during training in expectation (e.g. one epoch means that every example has been seen once). It is preferable to track epochs rather than iterations since the number of iterations depends on the arbitrary setting of batch size.

<a name='loss'></a>
#### Loss function

The first quantity that is useful to track during training is the loss, as it is evaluated on the individual batches during the forward pass. Below is a cartoon diagram showing the loss over time, and especially what the shape might tell you about the learning rate:

<div class="fig figcenter fighighlight">
  <img src="{{site.baseurl}}/assets/nn3/learningrates.jpeg" width="49%">
  <img src="{{site.baseurl}}/assets/nn3/loss.jpeg" width="49%">
  <div class="figcaption">
    <b>Left:</b> A cartoon depicting the effects of different learning rates. With low learning rates the improvements will be linear. With high learning rates they will start to look more exponential. Higher learning rates will decay the loss faster, but they get stuck at worse values of loss (green line). This is because there is too much "energy" in the optimization and the parameters are bouncing around chaotically, unable to settle in a nice spot in the optimization landscape. <b>Right:</b> An example of a typical loss function over time, while training a small network on CIFAR-10 dataset. This loss function looks reasonable (it might indicate a slightly too small learning rate based on its speed of decay, but it's hard to say), and also indicates that the batch size might be a little too low (since the cost is a little too noisy).
  </div>
</div>

The amount of "wiggle" in the loss is related to the batch size. When the batch size is 1, the wiggle will be relatively high. When the batch size is the full dataset, the wiggle will be minimal because every gradient update should be improving the loss function monotonically (unless the learning rate is set too high).

Some people prefer to plot their loss functions in the log domain. Since learning progress generally takes an exponential form shape, the plot appears more as a slightly more interpretable straight line, rather than a hockey stick. Additionally, if multiple cross-validated models are plotted on the same loss graph, the differences between them become more apparent.

Sometimes loss functions can look funny [lossfunctions.tumblr.com](http://lossfunctions.tumblr.com/).

<a name='accuracy'></a>
#### Train/Val accuracy

The second important quantity to track while training a classifier is the validation/training accuracy. This plot can give you valuable insights into the amount of overfitting in your model:

<div class="fig figleft fighighlight">
  <img src="{{site.baseurl}}/assets/nn3/accuracies.jpeg">
  <div class="figcaption">
    The gap between the training and validation accuracy indicates the amount of overfitting. Two possible cases are shown in the diagram on the left. The blue validation error curve shows very small validation accuracy compared to the training accuracy, indicating strong overfitting (note, it's possible for the validation accuracy to even start to go down after some point). When you see this in practice you probably want to increase regularization (stronger L2 weight penalty, more dropout, etc.) or collect more data. The other possible case is when the validation accuracy tracks the training accuracy fairly well. This case indicates that your model capacity is not high enough: make the model larger by increasing the number of parameters.
  </div>
  <div style="clear:both"></div>
</div>

<a name='ratio'></a>
#### Ratio of weights:updates

The last quantity you might want to track is the ratio of the update magnitudes to to the value magnitudes. Note: *updates*, not the raw gradients (e.g. in vanilla sgd this would be the gradient multiplied by the learning rate). You might want to evaluate and track this ratio for every set of parameters independently. A rough heuristic is that this ratio should be somewhere around 1e-3. If it is lower than this then the learning rate might be too low. If it is higher then the learning rate is likely too high. Here is a specific example:

~~~python
# assume parameter vector W and its gradient vector dW
param_scale = np.linalg.norm(W.ravel())
update = -learning_rate*dW # simple SGD update
update_scale = np.linalg.norm(update.ravel())
W += update # the actual update
print update_scale / param_scale # want ~1e-3
~~~

Instead of tracking the min or the max, some people prefer to compute and track the norm of the gradients and their updates instead. These metrics are usually correlated and often give approximately the same results.

<a name='distr'></a>
#### Activation / Gradient distributions per layer

An incorrect initialization can slow down or even completely stall the learning process. Luckily, this issue can be diagnosed relatively easily. One way to do so is to plot activation/gradient histograms for all layers of the network. Intuitively, it is not a good sign to see any strange distributions - e.g. with tanh neurons we would like to see a distribution of neuron activations between the full range of [-1,1], instead of seeing all neurons outputting zero, or all neurons being completely saturated at either -1 or 1.


<a name='vis'></a>
#### First-layer Visualizations

Lastly, when one is working with image pixels it can be helpful and satisfying to plot the first-layer features visually:

<div class="fig figcenter fighighlight">
  <img src="{{site.baseurl}}/assets/nn3/weights.jpeg" width="43%" style="margin-right:10px;">
  <img src="{{site.baseurl}}/assets/nn3/cnnweights.jpg" width="49%">
  <div class="figcaption">
    Examples of visualized weights for the first layer of a neural network. <b>Left</b>: Noisy features indicate could be a symptom: Unconverged network, improperly set learning rate, very low weight regularization penalty. <b>Right:</b> Nice, smooth, clean and diverse features are a good indication that the training is proceeding well.
  </div>
</div>

<a name='update'></a>
### Parameter updates

Once the analytic gradient is computed with backpropagation, the gradients are used to perform a parameter update. There are several approaches for performing the update, which we discuss next.

We note that optimization for deep networks is currently a very active area of research. In this section we highlight some established and common techniques you may see in practice, briefly describe their intuition, but leave a detailed analysis outside of the scope of the class. We provide some further pointers for an interested reader.

<a name='sgd'></a>
#### SGD and bells and whistles

**Vanilla update**. The simplest form of update is to change the parameters along the negative gradient direction (since the gradient indicates the direction of increase, but we usually wish to minimize a loss function). Assuming a vector of parameters `x` and the gradient `dx`, the simplest update has the form:

~~~python
# Vanilla update
x += - learning_rate * dx
~~~

where `learning_rate` is a hyperparameter - a fixed constant. When evaluated on the full dataset, and when the learning rate is low enough, this is guaranteed to make non-negative progress on the loss function.

**Momentum update** is another approach that almost always enjoys better converge rates on deep networks. This update can be motivated from a physical perspective of the optimization problem. In particular, the loss can be interpreted as a the height of a hilly terrain (and therefore also to the potential energy since $U = mgh$ and therefore $ U \propto h $ ). Initializing the parameters with random numbers is equivalent to setting a particle with zero initial velocity at some location. The optimization process can then be seen as equivalent to the process of simulating the parameter vector (i.e. a particle) as rolling on the landscape.

Since the force on the particle is related to the gradient of potential energy (i.e. $F = - \nabla U $ ), the **force** felt by the particle is precisely the (negative) **gradient** of the loss function. Moreover, $F = ma $ so the (negative) gradient is in this view proportional to the acceleration of the particle. Note that this is different from the SGD update shown above, where the gradient directly integrates the position. Instead, the physics view suggests an update in which the gradient only directly influences the velocity, which in turn has an effect on the position:

~~~python
# Momentum update
v = mu * v - learning_rate * dx # integrate velocity
x += v # integrate position
~~~

Here we see an introduction of a `v` variable that is initialized at zero, and an additional hyperparameter (`mu`). As an unfortunate misnomer, this variable is in optimization referred to as *momentum* (its typical value is about 0.9), but its physical meaning is more consistent with the coefficient of friction. Effectively, this variable damps the velocity and reduces the kinetic energy of the system, or otherwise the particle would never come to a stop at the bottom of a hill. When cross-validated, this parameter is usually set to values such as [0.5, 0.9, 0.95, 0.99]. Similar to annealing schedules for learning rates (discussed later, below), optimization can sometimes benefit a little from momentum schedules, where the momentum is increased in later stages of learning. A typical setting is to start with momentum of about 0.5 and anneal it to 0.99 or so over multiple epochs.

> With Momentum update, the parameter vector will build up velocity in any direction that has consistent gradient.

**Nesterov Momentum** is a slightly different version of the momentum update has recently been gaining popularity. It enjoys stronger theoretical converge guarantees for convex functions and in practice it also consistenly works slightly better than standard momentum.

The core idea behind Nesterov momentum is that when the current parameter vector is at some position `x`, then looking at the momentum update above, we know that the momentum term alone (i.e. ignoring the second term with the gradient) is about to nudge the parameter vector by `mu * v`. Therefore, if we are about to compute the gradient, we can treat the future approximate position `x + mu * v` as a "lookahead" - this is a point in the vicinity of where we are soon going to end up. Hence, it makes sense to compute the gradient at `x + mu * v` instead of at the "old/stale" position `x`. 

<div class="fig figcenter fighighlight">
  <img src="{{site.baseurl}}/assets/nn3/nesterov.jpeg">
  <div class="figcaption">
    Nesterov momentum. Instead of evaluating gradient at the current position (red circle), we know that our momentum is about to carry us to the tip of the green arrow. With Nesterov momentum we therefore instead evaluate the gradient at this "looked-ahead" position.
  </div>
</div>

That is, in a slightly awkward notation, we would like to do the following:

~~~python
x_ahead = x + mu * v
# evaluate dx_ahead (the gradient at x_ahead instead of at x)
v = mu * v - learning_rate * dx_ahead
x += v
~~~

However, in practice people prefer to express the update to look as similar to vanilla SGD or to the previous momentum update as possible. This is possible to achieve by manipulating the update above with a variable transform `x_ahead = x + mu * v`, and then expressing the update in terms of `x_ahead` instead of `x`. That is, the parameter vector we are actually storing is always the ahead version. The equations in terms of `x_ahead` (but renaming it back to `x`) then become:

~~~python
v_prev = v # back this up
v = mu * v - learning_rate * dx # velocity update stays the same
x += -mu * v_prev + (1 + mu) * v # position update changes form
~~~

We recommend this further reading to understand the source of these equations and the mathematical formulation of Nesterov's Accelerated Momentum (NAG):

- [Advances in optimizing Recurrent Networks](http://arxiv.org/pdf/1212.0901v2.pdf) by Yoshua Bengio, Section 3.5.
- [Ilya Sutskever's thesis](http://www.cs.utoronto.ca/~ilya/pubs/ilya_sutskever_phd_thesis.pdf) (pdf) contains a longer exposition of the topic in section 7.2


<a name='anneal'></a>
#### Annealing the learning rate

In training deep networks, it is usually helpful to anneal the learning rate over time. Good intuition to have in mind is that with a high learning rate, the system contains too much kinetic energy and the parameter vector bounces around chaotically, unable to settle down into deeper, but narrower parts of the loss function. Knowing when to decay the learning rate can be tricky: Decay it slowly and you'll be wasting computation bouncing around chaotically with little improvement for a long time. But decay it too aggressively and the system will cool too quickly, unable to reach the best position it can. There are three common types of implementing the learning rate decay:

- **Step decay**: Reduce the learning rate by some factor every few epochs. Typical values might be reducing the learning rate by a half every 5 epochs, or by 0.1 every 20 epochs. These numbers depend heavily on the type of problem and the model. One heuristic you may see in practice is to watch the validation error while training with a fixed learning rate, and reduce the learning rate by a constant (e.g. 0.5) whenever the validation error stops improving.
- **Exponential decay.** has the mathematical form $\alpha = \alpha_0 e^{-k t}$, where $\alpha_0, k$ are hyperparameters and $t$ is the iteration number (but you can also use units of epochs).
- **1/t decay** has the mathematical form $\alpha = \alpha_0 / (1 + k t )$ where $a_0, k$ are hyperparameters and $t$ is the iteration number.

In practice, we find that the step decay dropout is slightly preferable because the hyperparameters it involves (the fraction of decay and the step timings in units of epochs) are more interpretable than the hyperparameter $k$. Lastly, if you can afford the computational budget, err on the side of slower decay and train for a longer time.

<a name='second'></a>
#### Second order methods

A second, popular group of methods for optimization in context of deep learning is based on [Newton's method](http://en.wikipedia.org/wiki/Newton%27s_method_in_optimization), which iterates the following update:

$$
x \leftarrow x - [H f(x)]^{-1} \nabla f(x)
$$

Here, $H f(x)$ is the [Hessian matrix](http://en.wikipedia.org/wiki/Hessian_matrix), which is a square matrix of second-order partial derivatives of the function. The term $\nabla f(x)$ is the gradient vector, as seen in Gradient Descent. Intuitively, the Hessian describes the local curvature of the loss function, which allows us to perform a more efficient update. In particular, multiplying by the inverse Hessian leads the optimization to take more aggressive steps in directions of shallow curvature and shorter steps in directions of steep curvature. Note, crucially, the absence of any learning rate hyperparameters in the update formula, which the proponents of these methods cite this as a large advantage over first-order methods.

However, the update above is impractical for most deep learning applications because computing (and inverting) the Hessian in its explicit form is a very costly process in both space and time. For instance, a Neural Network with one million parameters would have a Hessian matrix of size [1,000,000 x 1,000,000], occupying approximately 3725 gigabytes of RAM. Hence, a large variety of *quasi-Newton* methods have been developed that seek to approximate the inverse Hessian. Among these, the most popular is [L-BFGS](http://en.wikipedia.org/wiki/Limited-memory_BFGS), which uses the information in the gradients over time to form the approximation implicitly (i.e. the full matrix is never computed).

However, even after we eliminate the memory concerns, a large downside of a naive application of L-BFGS is that it must be computed over the entire training set, which could contain millions of examples. Unlike mini-batch SGD, getting L-BFGS to work on mini-batches is more tricky and an active area of research.

**In practice**, it is currently not common to see L-BFGS or similar second-order methods applied to large-scale Deep Learning and Convolutional Neural Networks. Instead, SGD variants based on (Nesterov's) momentum are more standard because they are simpler and scale more easily.

Additional references:

- [Large Scale Distributed Deep Networks](http://research.google.com/archive/large_deep_networks_nips2012.html) is a paper from the Google Brain team, comparing L-BFGS and SGD variants in large-scale distributed optimization.
- [SFO](http://arxiv.org/abs/1311.2115) algorithm strives to combine the advantages of SGD with advantages of L-BFGS.

<a name='ada'></a>
#### Per-parameter adaptive learning rate methods

All previous approaches we've discussed so far manipulated the learning rate globally and equally for all parameters. Tuning the learning rates is an expensive process, so much work has gone into devising methods that can adaptively tune the learning rates, and even do so per parameter. Many of these methods may still require other hyperparameter settings, but the argument is that they are well-behaved for a broader range of hyperparameter values than the raw learning rate. In this section we highlight some common adaptive methods you may encounter in practice:

**Adagrad** is an adaptive learning rate method originally proposed by [Duchi et al.](http://jmlr.org/papers/v12/duchi11a.html).

~~~python
# Assume the gradient dx and parameter vector x
cache += dx**2
x += - learning_rate * dx / (np.sqrt(cache) + eps)
~~~

Notice that the variable `cache` has size equal to the size of the gradient, and keeps track of per-parameter sum of squared gradients. This is then used to normalize the parameter update step, element-wise. Notice that the weights that receive high gradients will have their effective learning rate reduced, while weights that receive small or infrequent updates will have their effective learning rate increased. Amusingly, the square root operation turns out to be very important and without it the algorithm performs much worse. The smoothing term `eps` (usually set somewhere in range from 1e-4 to 1e-8) avoids division by zero. A downside of Adagrad is that in case of Deep Learning, the monotonic learning rate usually proves too aggressive and stops learning too early.

**RMSprop.** RMSprop is a very effective, but currently unpublished adaptive learning rate method. Amusingly, everyone who uses this method in their work currently cites [slide 29 of Lecture 6](http://www.cs.toronto.edu/~tijmen/csc321/slides/lecture_slides_lec6.pdf) of Geoff Hinton's Coursera class. The RMSProp update adjusts the Adagrad method in a very simple way in an attempt to reduce its aggressive, monotonically decreasing learning rate. In particular, it uses a moving average of squared gradients instead, giving:

~~~python
cache = decay_rate * cache + (1 - decay_rate) * dx**2
x += - learning_rate * dx / (np.sqrt(cache) + eps)
~~~

Here, `decay_rate` is a hyperparameter and typical values are [0.9, 0.99, 0.999]. Notice that the `x+=` update is identical to Adagrad, but the `cache` variable is a "leaky". Hence, RMSProp still modulates the learning rate of each weight based on the magnitudes of its gradients, which has a beneficial equalizing effect, but unlike Adagrad the updates do not get monotonically smaller.

**Adam.** [Adam](http://arxiv.org/abs/1412.6980) is a recently proposed update that looks a bit like RMSProp with momentum. The (simplified) update looks as follows:

~~~python
m = beta1*m + (1-beta1)*dx
v = beta2*v + (1-beta2)*(dx**2)
x += - learning_rate * m / (np.sqrt(v) + eps)
~~~

Notice that the update looks exactly as RMSProp update, except the "smooth" version of the gradient `m` is used instead of the raw (and perhaps noisy) gradient vector `dx`. Recommended values in the paper are `eps = 1e-8`, `beta1 = 0.9`, `beta2 = 0.999`. In practice Adam is currently recommended as the default algorithm to use, and often works slightly better than RMSProp. However, it is often also worth trying SGD+Nesterov Momentum as an alternative. The full Adam update also includes a *bias correction* mechanism, which compensates for the fact that in the first few time steps the vectors `m,v` are both initialized and therefore biased at zero, before they fully "warm up". We refer the reader to the paper for the details, or the course slides where this is expanded on.

Additional References:

- [Unit Tests for Stochastic Optimization](http://arxiv.org/abs/1312.6055) proposes a series of tests as a standardized benchmark for stochastic optimization.

<div class="fig figcenter fighighlight">
  <img src="{{site.baseurl}}/assets/nn3/opt2.gif" width="49%" style="margin-right:10px;">
  <img src="{{site.baseurl}}/assets/nn3/opt1.gif" width="49%">
  <div class="figcaption">
    Animations that may help your intuitions about the learning process dynamics. <b>Left:</b> Contours of a loss surface and time evolution of different optimization algorithms. Notice the "overshooting" behavior of momentum-based methods, which make the optimization look like a ball rolling down the hill. <b>Right:</b> A visualization of a saddle point in the optimization landscape, where the curvature along different dimension has different signs (one dimension curves up and another down). Notice that SGD has a very hard time breaking symmetry and gets stuck on the top. Conversely, algorithms such as RMSprop will see very low gradients in the saddle direction. Due to the denominator term in the RMSprop update, this will increase the effective learning rate along this direction, helping RMSProp proceed. Images credit: <a href="https://twitter.com/alecrad">Alec Radford</a>.
  </div>
</div>

<a name='hyper'></a>
### Hyperparameter optimization

As we've seen, training Neural Networks can involve many hyperparameter settings. The most common hyperparameters in context of Neural Networks include:

- the initial learning rate
- learning rate decay schedule (such as the decay constant)
- regularization strength (L2 penalty, dropout strength)

But as saw, there are many more relatively less sensitive hyperparameters, for example in per-parameter adaptive learning methods, the setting of momentum and its schedule, etc. In this section we describe some additional tips and tricks for performing the hyperparameter search:

**Implementation**. Larger Neural Networks typically require a long time to train, so performing hyperparameter search can take many days/weeks. It is important to keep this in mind since it influences the design of your code base. One particular design is to have a **worker** that continuously samples random hyperparameters and performs the optimization. During the training, the worker will keep track of the validation performance after every epoch, and writes a model checkpoint (together with miscellaneous training statistics such as the loss over time) to a file, preferably on a shared file system. It is useful to include the validation performance directly in the filename, so that it is simple to inspect and sort the progress. Then there is a second program which we will call a **master**, which launches or kills workers across a computing cluster, and may additionally inspect the checkpoints written by workers and plot their training statistics, etc.

**Prefer one validation fold to cross-validation**. In most cases a single validation set of respectable size substantially simplifies the code base, without the need for cross-validation with multiple folds. You'll hear people say they "cross-validated" a parameter, but many times it is assumed that they still only used a single validation set.

**Hyperparameter ranges**. Search for hyperparameters on log scale. For example, a typical sampling of the learning rate would look as follows: `learning_rate = 10 ** uniform(-6, 1)`. That is, we are generating a random random with a uniform distribution, but then raising it to the power of 10. The same strategy should be used for the regularization strength. Intuitively, this is because learning rate and regularization strength have multiplicative effects on the training dynamics. For example, a fixed change of adding 0.01 to a learning rate has huge effects on the dynamics if the learning rate is 0.001, but nearly no effect if the learning rate when it is 10. This is because the learning rate multiplies the computed gradient in the update. Therefore, it is much more natural to consider a range of learning rate multiplied or divided by some value, than a range of learning rate added or subtracted to by some value. Some parameters (e.g. dropout) are instead usually searched in the original scale (e.g. `dropout = uniform(0,1)`).

**Prefer random search to grid search**. As argued by Bergstra and Bengio in [Random Search for Hyper-Parameter Optimization](http://www.jmlr.org/papers/volume13/bergstra12a/bergstra12a.pdf), "randomly chosen trials are more efficient for hyper-parameter optimization than trials on a grid". As it turns out, this is also usually easier to implement.

<div class="fig figcenter fighighlight">
  <img src="{{site.baseurl}}/assets/nn3/gridsearchbad.jpeg" width="50%">
  <div class="figcaption">
    Core illustration from <a href="http://www.jmlr.org/papers/volume13/bergstra12a/bergstra12a.pdf">Random Search for Hyper-Parameter Optimization</a> by Bergstra and Bengio. It is very often the case that some of the hyperparameters matter much more than others (e.g. top hyperparam vs. left one in this figure). Performing random search rather than grid search allows you to much more precisely discover good values for the important ones.
  </div>
</div>

**Careful with best values on border**. Sometimes it can happen that you're searching for a hyperparameter (e.g. learning rate) in a bad range. For example, suppose we use `learning_rate = 10 ** uniform(-6, 1)`. Once we receive the results, it is important to double check that the final learning rate is not at the edge of this interval, or otherwise you may be missing more optimal hyperparameter setting beyond the interval.

**Stage your search from coarse to fine**. In practice, it can be helpful to first search in coarse ranges (e.g. 10 ** [-6, 1]), and then depending on where the best results are turning up, narrow the range. Also, it can be helpful to perform the initial coarse search while only training for 1 epoch or even less, because many hyperparameter settings can lead the model to not learn at all, or immediately explode with infinite cost. The second stage could then perform a narrower search with 5 epochs, and the last stage could perform a detailed search in the final range for many more epochs (for example).

**Bayesian Hyperparameter Optimization** is a whole area of research devoted to coming up with algorithms that try to more efficiently navigate the space of hyperparameters. The core idea is to appropriately balance the exploration - exploitation trade-off when querying the performance at different hyperparameters. Multiple libraries have been developed based on these models as well, among some of the better known ones are [Spearmint](https://github.com/JasperSnoek/spearmint), [SMAC](http://www.cs.ubc.ca/labs/beta/Projects/SMAC/), and [Hyperopt](http://jaberg.github.io/hyperopt/). However, in practical settings with ConvNets it is still relatively difficult to beat random search in a carefully-chosen intervals. See some additional from-the-trenches discussion [here](http://nlpers.blogspot.com/2014/10/hyperparameter-search-bayesian.html).

<a name='eval'></a>
## Evaluation

<a name='ensemble'></a>
### Model Ensembles

In practice, one reliable approach to improving the performance of Neural Networks by a few percent is to train multiple independent models, and at test time average their predictions. As the number of models in the ensemble increases, the performance typically monotonically improves (though with diminishing returns). Moreover, the improvements are more dramatic with higher model variety in the ensemble. There are a few approaches to forming an ensemble:

- **Same model, different initializations**. Use cross-validation to determine the best hyperparameters, then train multiple models with the best set of hyperparameters but with different random initialization. The danger with this approach is that the variety is only due to initialization.
- **Top models discovered during cross-validation**. Use cross-validation to determine the best hyperparameters, then pick the top few (e.g. 10) models to form the ensemble. This improves the variety of the ensemble but has the danger of including suboptimal models. In practice, this can be easier to perform since it doesn't require additional retraining of models after cross-validation
- **Different checkpoints of a single model**. If training is very expensive, some people have had limited success in taking different checkpoints of a single network over time (for example after every epoch) and using those to form an ensemble. Clearly, this suffers from some lack of variety, but can still work reasonably well in practice. The advantage of this approach is that is very cheap.
- **Running average of parameters during training**. Related to the last point, a cheap way of almost always getting an extra percent or two of performance is to maintain a second copy of the network's weights in memory that maintains an exponentially decaying sum of previous weights during training. This way you're averaging the state of the network over last several iterations. You will find that this "smoothed" version of the weights over last few steps almost always achieves better validation error. The rough intuition to have in mind is that the objective is bowl-shaped and your network is jumping around the mode, so the average has a higher chance of being somewhere nearer the mode.

One disadvantage of model ensembles is that they take longer to evaluate on test example. An interested reader may find the recent work from Geoff Hinton on ["Dark Knowledge"](https://www.youtube.com/watch?v=EK61htlw8hY) inspiring, where the idea is to "distill" a good ensemble back to a single model by incorporating the ensemble log likelihoods into a modified objective.

<a name='summary'></a>
## Summary

To train a Neural Network:

- Gradient check your implementation with a small batch of data and be aware of the pitfalls.
- As a sanity check, make sure your initial loss is reasonable, and that you can achieve 100% training accuracy on a very small portion of the data
- During training, monitor the loss, the training/validation accuracy, and if you're feeling fancier, the magnitude of updates in relation to parameter values (it should be ~1e-3), and when dealing with ConvNets, the first-layer weights.
- The two recommended updates to use are either SGD+Nesterov Momentum or Adam.
- Decay your learning rate over the period of the training. For example, halve the learning rate after a fixed number of epochs, or whenever the validation accuracy tops off.
- Search for good hyperparameters with random search (not grid search). Stage your search from coarse (wide hyperparameter ranges, training only for 1-5 epochs), to fine (narrower rangers, training for many more epochs)
- Form model ensembles for extra performance

<a name='add'></a>
## Additional References

- [SGD](http://research.microsoft.com/pubs/192769/tricks-2012.pdf) tips and tricks from Leon Bottou
- [Efficient BackProp](http://yann.lecun.com/exdb/publis/pdf/lecun-98b.pdf) (pdf) from Yann LeCun
- [Practical Recommendations for Gradient-Based Training of Deep
Architectures](http://arxiv.org/pdf/1206.5533v2.pdf) from Yoshua Bengio

---
<p style="text-align:right"><b>
번역: 최영근 <a href="https://github.com/ygchoistat" style="color:black">ygchoistat</a>
</b></p>
