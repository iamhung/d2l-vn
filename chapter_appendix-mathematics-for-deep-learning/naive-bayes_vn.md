<!-- ===================== Bắt đầu dịch Phần 1 ==================== -->
<!-- ========================================= REVISE - BẮT ĐẦU =================================== -->

<!--
# Naive Bayes
-->

# Bộ phân loại Naive Bayes
:label:`sec_naive_bayes`


<!--
Throughout the previous sections, we learned about the theory of probability and random variables.
To put this theory to work, let us introduce the *naive Bayes* classifier.
This uses nothing but probabilistic fundamentals to allow us to perform classification of digits.
-->

Xuyên suốt các phần trước, chúng ta đã học về lý thuyết xác suất và các biến ngẫu nhiên.
Để áp dụng lý thuyết này vào thực tiễn, chúng tôi giới thiệu về bộ phân loại *naive Bayes*.
Phương pháp này không sử dụng bất kỳ điều gì ngoài các lý thuyết căn bản về xác suất nhằm cho phép chúng ta thực hiện phân loại các chữ số.


<!--
Learning is all about making assumptions. If we want to classify a new data example 
that we have never seen before we have to make some assumptions about which data examples are similar to each other.
The naive Bayes classifier, a popular and remarkably clear algorithm, assumes all features are independent from each other to simplify the computation.
In this section, we will apply this model to recognize characters in images.
-->

Quá trình học hoàn toàn xoay quanh việc đưa ra các giả định. Nếu chúng ta muốn phân loại một mẫu dữ liệu mới
mà ta chưa bao giờ thấy trước đó, ta cần phải đưa ra một giả định nào đó cho câu hỏi những mẫu dữ liệu nào giống nhau.
Bộ phân loại Naive Bayes này, một thuật toán thông dụng và khá dễ hiểu, giả định rằng tất cả các đặc trưng đều độc lập với nhau để đơn giản hóa việc tính toán.
Trong phần này, chúng tôi sẽ áp dụng mô hình đó để nhận dạng các ký tự trong ảnh.


```{.python .input}
%matplotlib inline
from d2l import mxnet as d2l
import math
from mxnet import gluon, np, npx
npx.set_np()
d2l.use_svg_display()
```

```{.python .input}
#@tab pytorch
%matplotlib inline
from d2l import torch as d2l
import math
import torch
import torchvision
d2l.use_svg_display()
```

```{.python .input}
#@tab tensorflow
%matplotlib inline
from d2l import tensorflow as d2l
import math
import tensorflow as tf
d2l.use_svg_display()
```


<!--
## Optical Character Recognition
-->

## Nhận diện Ký tự Quang học


<!--
MNIST :cite:`LeCun.Bottou.Bengio.ea.1998` is one of widely used datasets.
It contains 60,000 images for training and 10,000 images for validation.
Each image contains a handwritten digit from 0 to 9.
The task is classifying each image into the corresponding digit.
-->

MNIST :cite:`LeCun.Bottou.Bengio.ea.1998` là một trong các tập dữ liệu được sử dụng rộng rãi.
Nó chứa 60.000 hình ảnh để huấn luyện và 10.000 hình ảnh để kiểm định.
Mỗi hình ảnh chứa một chữ số viết tay từ 0 đến 9.
Nhiệm vụ là phân loại từng hình ảnh với chữ số tương ứng.


<!--
Gluon provides a `MNIST` class in the `data.vision` module to
automatically retrieve the dataset from the Internet.
Subsequently, Gluon will use the already-downloaded local copy.
We specify whether we are requesting the training set or the test set
by setting the value of the parameter `train` to `True` or `False`, respectively.
Each image is a grayscale image with both width and height of $28$ with shape ($28$,$28$,$1$).
We use a customized transformation to remove the last channel dimension.
In addition, the dataset represents each pixel by an unsigned $8$-bit integer.
We quantize them into binary features to simplify the problem.
-->

Gluon cung cấp một lớp `MNIST` trong mô-đun `data.vision` để
tự động truy xuất tập dữ liệu từ Internet. 
Sau đó, Gluon sẽ sử dụng bản sao cục bộ đã được tải xuống.
Chúng ta chỉ định rằng ta đang yêu cầu tập huấn luyện hay tập kiểm tra
bằng cách đặt giá trị của tham số `train` thành` True` hoặc `False` tương ứng.
Mỗi hình ảnh là một hình ảnh thang màu xám có cả chiều rộng và chiều cao là $28$ với kích thước ($28$,$28$,$1$).
Ta sẽ sử dụng một phép biến đổi tùy chỉnh để loại bỏ chiều kênh cuối cùng.
Ngoài ra, tập dữ liệu biểu diễn mỗi điểm ảnh bằng một số nguyên $8$-bit không dấu.
Ta lượng hóa (*quantize*) chúng thành các đặc trưng nhị phân để đơn giản hóa vấn đề.



```{.python .input}
def transform(data, label):
    return np.floor(data.astype('float32') / 128).squeeze(axis=-1), label

mnist_train = gluon.data.vision.MNIST(train=True, transform=transform)
mnist_test = gluon.data.vision.MNIST(train=False, transform=transform)
```

```{.python .input}
#@tab pytorch
data_transform = torchvision.transforms.Compose(
    [torchvision.transforms.ToTensor()])

mnist_train = torchvision.datasets.MNIST(
    root='./temp', train=True, transform=data_transform, download=True)
mnist_test = torchvision.datasets.MNIST(
    root='./temp', train=False, transform=data_transform, download=True)
```

```{.python .input}
#@tab tensorflow
(train_images, train_labels), (test_images, test_labels) = tf.keras.datasets.mnist.load_data()
```


<!--
We can access a particular example, which contains the image and the corresponding label.
-->

Ta có thể truy cập vào từng mẫu cụ thể, có chứa hình ảnh và nhãn tương ứng.


```{.python .input}
image, label = mnist_train[2]
image.shape, label
```

```{.python .input}
#@tab pytorch
image, label = mnist_train[2]
image.shape, label
```

```{.python .input}
#@tab tensorflow
image, label = train_images[2], train_labels[2]
image.shape, label
```


<!--
Our example, stored here in the variable `image`, corresponds to an image with a height and width of $28$ pixels.
-->

Ví dụ của ta ở đây được lưu trữ ở đây trong biến `image`, tương ứng với một hình ảnh có chiều cao và chiều rộng là $28$ pixel.


```{.python .input}
#@tab all
image.shape, image.dtype
```


<!--
Our code stores the label of each image as a scalar. Its type is a $32$-bit integer.
-->

Đoạn mã của chúng ta lưu trữ nhãn của mỗi hình ảnh dưới dạng số vô hướng với kiểu dữ liệu là số nguyên $32$-bit.


```{.python .input}
label, type(label), label.dtype
```

```{.python .input}
#@tab pytorch
label, type(label)
```

```{.python .input}
#@tab tensorflow
label, type(label)
```


<!--
We can also access multiple examples at the same time.
-->

Ta cũng có thể truy cập vào nhiều mẫu cùng một lúc.


```{.python .input}
images, labels = mnist_train[10:38]
images.shape, labels.shape
```

```{.python .input}
#@tab pytorch
images = torch.stack([mnist_train[i][0] for i in range(10,38)], 
                     dim=1).squeeze(0)
labels = torch.tensor([mnist_train[i][1] for i in range(10,38)])
images.shape, labels.shape
```

```{.python .input}
#@tab tensorflow
images = tf.stack([train_images[i] for i in range(10, 38)], axis=0)
labels = tf.constant([train_labels[i] for i in range(10, 38)])
images.shape, labels.shape
```


<!--
Let us visualize these examples.
-->

Hãy cùng minh họa các mẫu sau.


```{.python .input}
#@tab all
d2l.show_images(images, 2, 9);
```

<!-- ===================== Kết thúc dịch Phần 1 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 2 ===================== -->

<!--
## The Probabilistic Model for Classification
-->

## Mô hình xác suất để Phân loại


<!--
In a classification task, we map an example into a category.
Here an example is a grayscale $28\times 28$ image, and a category is a digit.
(Refer to :numref:`sec_softmax` for a more detailed explanation.)
One natural way to express the classification task is via the probabilistic question: what is the most likely label given the features (i.e., image pixels)?
Denote by $\mathbf x\in\mathbb R^d$ the features of the example and $y\in\mathbb R$ the label.
Here features are image pixels, where we can reshape a $2$-dimensional image to a vector so that $d=28^2=784$, and labels are digits.
The probability of the label given the features is $p(y  \mid  \mathbf{x})$. If we are able to compute these probabilities, 
which are $p(y  \mid  \mathbf{x})$ for $y=0, \ldots,9$ in our example, then the classifier will output the prediction $\hat{y}$ given by the expression:
-->

Trong nhiệm vụ phân loại, chúng tôi ánh xạ một mẫu thành một danh mục.
Dưới đây là một ví dụ ảnh xám kích thước $28\times 28$ và hạng mục là một chữ số.
(Tham khảo :numref:`sec_softmax` để xem giải thích chi tiết hơn.)
Một cách tự nhiên để thể hiện tác vụ phân loại là thông qua câu hỏi xác suất: nhãn nào là hợp lý nhất với các đặc trưng cho trước (tức là các pixel trong ảnh)?
Ký hiệu $\mathbf x\in\mathbb R^d$  là các đặc trưng và $y\in\mathbb R$ là nhãn của một mẫu.
Đặc trưng ở đây là các pixel trong ảnh $2$ chiều mà ta có thể đổi kích thước thành một vector với $d=28^2=784$, và nhãn là các chữ số.
Xác suất của nhãn cho đặc trưng cho trước là $p(y  \mid  \mathbf{x})$. Nếu chúng ta có thể tính toán những xác suất này,
là $p(y  \mid  \mathbf{x})$ cho $y=0, \ldots,9$ trong mẫu của chúng ta, sau đó bộ phân loại sẽ xuất ra dự đoán $\hat{y}$ được đưa ra bởi biểu thức:



$$\hat{y} = \mathrm{argmax} \> p(y  \mid  \mathbf{x}).$$


<!--
Unfortunately, this requires that we estimate $p(y  \mid  \mathbf{x})$ for every value of $\mathbf{x} = x_1, ..., x_d$.
Imagine that each feature could take one of $2$ values.
For example, the feature $x_1 = 1$ might signify that the word apple appears in a given document and $x_1 = 0$ would signify that it does not.
If we had $30$ such binary features, that would mean that we need to be prepared to classify any of $2^{30}$ (over 1 billion!) possible values of the input vector $\mathbf{x}$.
-->

Rất tiếc, điều này yêu cầu ta ước tính $p(y  \mid  \mathbf{x})$ cho mọi giá trị của $\mathbf{x} = x_1, ..., x_d$.
Tưởng tượng rằng mỗi đặc trưng nhận lấy một trong các giá trị $2$.
Ví dụ, đặc trưng $x_1 = 1$ có thể biểu thị rằng từ "quả táo" xuất hiện trong một tài liệu nhất định và $x_1 = 0$ sẽ biểu thị rằng từ đó không xuất hiện.
Nếu chúng ta có $30$ đặc trưng nhị phân như vậy, điều đó có nghĩa là chúng ta cần phải chuẩn bị để phân loại bất kỳ giá trị nào trong số $2^{30}$ (hơn 1 tỷ!) các vector đầu khả dĩ của $\mathbf{x}$.


<!--
Moreover, where is the learning? If we need to see every single possible example in order to predict 
the corresponding label then we are not really learning a pattern but just memorizing the dataset.
-->

Hơn nữa, như vậy đâu có phải là học. Nếu chúng ta cần xem qua tất cả các ví dụ có thể có để dự đoán
nhãn tương ứng thì chúng ta không thực sự đang học một khuôn mẫu nào mà chỉ là đang ghi nhớ tập dữ liệu.


<!--
## The Naive Bayes Classifier
-->

## Bộ phân loại Naive Bayes


<!--
Fortunately, by making some assumptions about conditional independence, 
we can introduce some inductive bias and build a model capable of generalizing from a comparatively modest selection of training examples.
To begin, let us use Bayes theorem, to express the classifier as
-->

May mắn thay, bằng cách đưa ra một số giả định về tính độc lập có điều kiện,
ta có thể đưa vào một số thiên kiến quy nạp và xây dựng một mô hình có khả năng tổng quát hóa từ một nhóm các mẫu huấn luyện với kích thước tương đối khiêm tốn.
Để bắt đầu, ta hãy sử dụng định lý Bayes, để biểu thị bộ phân loại bằng biểu thức sau


$$\hat{y} = \mathrm{argmax}_y \> p(y  \mid  \mathbf{x}) = \mathrm{argmax}_y \> \frac{p( \mathbf{x}  \mid  y) p(y)}{p(\mathbf{x})}.$$


<!--
Note that the denominator is the normalizing term $p(\mathbf{x})$ which does not depend on the value of the label $y$.
As a result, we only need to worry about comparing the numerator across different values of $y$.
Even if calculating the denominator turned out to be intractable, we could get away with ignoring it, so long as we could evaluate the numerator.
Fortunately, even if we wanted to recover the normalizing constant, we could.
We can always recover the normalization term since $\sum_y p(y  \mid  \mathbf{x}) = 1$.
-->

Lưu ý rằng mẫu số là số hạng chuẩn hóa $p(\mathbf{x})$ không phụ thuộc vào giá trị của nhãn $y$.
Do đó, chúng ta chỉ cần quan tâm về việc so sánh tử số giữa các giá trị khác nhau của $y$.
Ngay cả khi việc tính toán mẫu số hóa ra là không thể, chúng ta có thể bỏ qua nó, miễn là chúng ta có thể tính được phần tử số.
May mắn thay, ta vẫn có thể khôi phục lại hằng số chuẩn hóa nếu ta muốn.
Ta luôn có thể khôi phục số hạng chuẩn hóa vì $\sum_y p(y  \mid  \mathbf{x}) = 1$.


<!--
Now, let us focus on $p( \mathbf{x}  \mid  y)$.
Using the chain rule of probability, we can express the term $p( \mathbf{x}  \mid  y)$ as
-->

Bây giờ, ta hãy tập trung vào biểu thức $p( \mathbf{x}  \mid  y)$.
Sử dụng quy tắc chuỗi cho xác suất, chúng ta có thể biểu thị số hạng $p( \mathbf{x}  \mid  y)$ dưới dạng toán học 


$$p(x_1  \mid y) \cdot p(x_2  \mid  x_1, y) \cdot ... \cdot p( x_d  \mid  x_1, ..., x_{d-1}, y).$$


<!--
By itself, this expression does not get us any further. We still must estimate roughly $2^d$ parameters.
However, if we assume that *the features are conditionally independent of each other, given the label*, 
then suddenly we are in much better shape, as this term simplifies to $\prod_i p(x_i  \mid  y)$, giving us the predictor
-->

Tự nó, biểu thức này không giúp ta được thêm điều gì. Ta vẫn phải ước tính khoảng $2^d$ các tham số.
Tuy nhiên, nếu chúng ta giả định rằng *các đặc trưng độc lập có điều kiện với nhau, với nhãn cho trước*,
thì đột nhiên ta đang ở trong tình trạng tốt hơn nhiều, vì số hạng này đơn giản hóa thành $\prod_i p(x_i  \mid  y)$, cho ta hàm dự đoán


$$ \hat{y} = \mathrm{argmax}_y \> \prod_{i=1}^d p(x_i  \mid  y) p(y).$$


<!--
If we can estimate $\prod_i p(x_i=1  \mid  y)$ for every $i$ and $y$, and save its value in $P_{xy}[i, y]$, 
here $P_{xy}$ is a $d\times n$ matrix with $n$ being the number of classes and $y\in\{1, \ldots, n\}$.
In addition, we estimate $p(y)$ for every $y$ and save it in $P_y[y]$, with $P_y$ a $n$-length vector.
Then for any new example $\mathbf x$, we could compute
-->

Nếu ta có thể ước lượng $\prod_i p(x_i=1  \mid  y)$ cho mỗi $i$ và $y$, và lưu giá trị của nó trong $P_{xy}[i, y]$, 
ở đây $P_{xy}$ là một ma trận có kích thước $d\times n$ với $n$ là số lượng các lớp và $y\in\{1, \ldots, n\}$.
Bổ sung thêm, ta ước lượng $p(y)$ cho mỗi $y$ và lưu nó trong $P_y[y]$, với $P_y$ là một vector có độ dài $n$.
Sau đó, đối với bất kỳ mẫu mới nào $\mathbf x$, ta có thể tính


$$ \hat{y} = \mathrm{argmax}_y \> \prod_{i=1}^d P_{xy}[x_i, y]P_y[y],$$
:eqlabel:`eq_naive_bayes_estimation`


<!--
for any $y$. So our assumption of conditional independence has taken the complexity of our model from 
an exponential dependence on the number of features $\mathcal{O}(2^dn)$ to a linear dependence, which is $\mathcal{O}(dn)$.
-->

cho $y$ bất kỳ. Vì vậy, giả định của chúng ta về sự độc lập có điều kiện đã làm giảm độ phức tạp của mô hình từ
phụ thuộc theo cấp số nhân vào số lượng các đặc trưng $\mathcal{O}(2^dn)$ thành phụ thuộc tuyến tính, tức là $\mathcal{O}(dn)$.

<!-- ===================== Kết thúc dịch Phần 2 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 3 ===================== -->

<!--
## Training
-->

## Huấn luyện


<!--
The problem now is that we do not know $P_{xy}$ and $P_y$.
So we need to estimate their values given some training data first.
This is *training* the model. Estimating $P_y$ is not too hard.
Since we are only dealing with $10$ classes, we may count the number of occurrences $n_y$ for each of the digits and divide it by the total amount of data $n$.
For instance, if digit 8 occurs $n_8 = 5,800$ times and we have a total of $n = 60,000$ images, the probability estimate is $p(y=8) = 0.0967$.
-->

Vấn đề bây giờ là ta không biết $P_{xy}$ và $P_y$.
Vì vậy, ta cần ước lượng giá trị của chúng với dữ liệu nào đó đã có.
Đây là *việc huấn luyện* mô hình. Ước lượng $P_y$ không quá khó.
Vì ta chỉ đang làm việc với $10$ lớp, ta có thể đếm số lần xuất hiện $n_y$ cho mỗi chữ số và chia nó cho tổng số dữ liệu $n$.
Chẳng hạn, nếu chữ số 8 xảy ra $n_8 = 5,800$ lần và ta có tổng số hình ảnh là $n = 60.000$, Ước lượng xác suất sẽ là $p(y=8) = 0.0967$.


```{.python .input}
X, Y = mnist_train[:]  # All training examples

n_y = np.zeros((10))
for y in range(10):
    n_y[y] = (Y == y).sum()
P_y = n_y / n_y.sum()
P_y
```

```{.python .input}
#@tab pytorch
X = torch.stack([mnist_train[i][0] for i in range(len(mnist_train))], 
                dim=1).squeeze(0)
Y = torch.tensor([mnist_train[i][1] for i in range(len(mnist_train))])

n_y = torch.zeros(10)
for y in range(10):
    n_y[y] = (Y == y).sum()
P_y = n_y / n_y.sum()
P_y
```

```{.python .input}
#@tab tensorflow
X = tf.stack([train_images[i] for i in range(len(train_images))], axis=0)
Y = tf.constant([train_labels[i] for i in range(len(train_labels))])

n_y = tf.Variable(tf.zeros(10))
for y in range(10):
    n_y[y].assign(tf.reduce_sum(tf.cast(Y == y, tf.float32)))
P_y = n_y / tf.reduce_sum(n_y)
P_y
```


<!--
Now on to slightly more difficult things $P_{xy}$. Since we picked black and white images, 
$p(x_i  \mid  y)$ denotes the probability that pixel $i$ is switched on for class $y$.
Just like before we can go and count the number of times $n_{iy}$ such that an event occurs and divide it by 
the total number of occurrences of $y$, i.e., $n_y$.
But there is something slightly troubling: certain pixels may never be black (e.g., for well cropped images the corner pixels might always be white).
A convenient way for statisticians to deal with this problem is to add pseudo counts to all occurrences.
Hence, rather than $n_{iy}$ we use $n_{iy}+1$ and instead of $n_y$ we use $n_{y} + 1$.
This is also called *Laplace Smoothing*. It may seem ad-hoc, however it may be well motivated from a Bayesian point-of-view.
-->

Giờ hãy chuyển sang vấn đề khó hơn một chút là tính $P_{xy}$. Vì ta lấy các ảnh đen trắng,
$p(x_i \mid y)$ biểu thị xác suất điểm ảnh $i$ được kích hoạt cho lớp $y$.
Đơn giản giống như trước đây, ta có thể duyệt và đếm số lần $n_{iy}$ để một sự kiện xảy ra và chia nó cho
tổng số lần xuất hiện của $y$, tức là $n_y$.
Nhưng có một điều hơi gây rắc rối: một số điểm ảnh nhất định có thể không bao giờ có màu đen (ví dụ: đối với các ảnh được cắt xén tốt, các điểm ảnh ở góc có thể luôn là màu trắng).
Một cách thuận tiện để các giải quyết vấn đề này là cộng thêm một số đếm giả vào tất cả các lần xuất hiện.
Do đó, thay vì $n_{iy} $, ta dùng $n_{iy} + 1$ và thay vì $n_y$, ta dùng $n_{y} + 1 $.
Phương pháp này còn được gọi là *Làm mượt Laplace* (*Laplace Smoothing*). Nó có vẻ không chính thống, tuy nhiên nó có thể được chào đón từ quan điểm Bayes.


```{.python .input}
n_x = np.zeros((10, 28, 28))
for y in range(10):
    n_x[y] = np.array(X.asnumpy()[Y.asnumpy() == y].sum(axis=0))
P_xy = (n_x + 1) / (n_y + 1).reshape(10, 1, 1)

d2l.show_images(P_xy, 2, 5);
```

```{.python .input}
#@tab pytorch
n_x = torch.zeros((10, 28, 28))
for y in range(10):
    n_x[y] = torch.tensor(X.numpy()[Y.numpy() == y].sum(axis=0))
P_xy = (n_x + 1) / (n_y + 1).reshape(10, 1, 1)

d2l.show_images(P_xy, 2, 5);
```

```{.python .input}
#@tab tensorflow
n_x = tf.Variable(tf.zeros((10, 28, 28)))
for y in range(10):
    n_x[y].assign(tf.cast(tf.reduce_sum(
        X.numpy()[Y.numpy() == y], axis=0), tf.float32))
P_xy = (n_x + 1) / tf.reshape((n_y + 1), (10, 1, 1))

d2l.show_images(P_xy, 2, 5);
```


<!--
By visualizing these $10\times 28\times 28$ probabilities (for each pixel for each class) we could get some mean looking digits.
-->


Bằng cách trực quan hóa các xác suất $10\times 28\times 28$ này (cho mỗi điểm ảnh đối với mỗi lớp), ta có thể lấy được hình ảnh trung bình của các chữ số.


<!--
Now we can use :eqref:`eq_naive_bayes_estimation` to predict a new image.
Given $\mathbf x$, the following functions computes $p(\mathbf x \mid y)p(y)$ for every $y$.
-->

Bây giờ ta có thể sử dụng :eqref:`eq_naive_bayes_estimation` để dự đoán một hình ảnh mới.
Cho $\mathbf x$, các hàm sau sẽ tính $p(\mathbf x \mid y)p(y)$ với mỗi $y$.

```{.python .input}
def bayes_pred(x):
    x = np.expand_dims(x, axis=0)  # (28, 28) -> (1, 28, 28)
    p_xy = P_xy * x + (1 - P_xy)*(1 - x)
    p_xy = p_xy.reshape(10, -1).prod(axis=1)  # p(x|y)
    return np.array(p_xy) * P_y

image, label = mnist_test[0]
bayes_pred(image)
```

```{.python .input}
#@tab pytorch
def bayes_pred(x):
    x = x.unsqueeze(0)  # (28, 28) -> (1, 28, 28)
    p_xy = P_xy * x + (1 - P_xy)*(1 - x)
    p_xy = p_xy.reshape(10, -1).prod(dim=1)  # p(x|y)
    return p_xy * P_y

image, label = mnist_test[0]
bayes_pred(image)
```

```{.python .input}
#@tab tensorflow
def bayes_pred(x):
    x = tf.expand_dims(x, axis=0)  # (28, 28) -> (1, 28, 28)
    p_xy = P_xy * x + (1 - P_xy)*(1 - x)
    p_xy = tf.math.reduce_prod(tf.reshape(p_xy, (10, -1)), axis=1)  # p(x|y)
    return p_xy * P_y

image, label = tf.cast(train_images[0], tf.float32), train_labels[0]
bayes_pred(image)
```


<!--
This went horribly wrong! To find out why, let us look at the per pixel probabilities.
They are typically numbers between $0.001$ and $1$. We are multiplying $784$ of them.
At this point it is worth mentioning that we are calculating these numbers on a computer, hence with a fixed range for the exponent.
What happens is that we experience *numerical underflow*, i.e., multiplying all the small numbers leads to something even smaller until it is rounded down to zero.
We discussed this as a theoretical issue in :numref:`sec_maximum_likelihood`, but we see the phenomena clearly here in practice.
-->

Điều này đã dẫn tới sai lầm khủng khiếp! Để tìm hiểu lý do tại sao, ta hãy xem xét xác suất trên mỗi điểm ảnh.
Chúng thường là những con số từ $0.001$ đến $1$ và ta đang nhân chúng $784$ lần.
Tại điểm này, điều đáng nói là ta đang tính những con số này trên máy tính, do đó với một phạm vi cố định cho số mũ.
Điều xảy ra là chúng ta gặp phải *rò rỉ số (underflow)*, tức là tích tất cả các số nhỏ hơn một sẽ dẫn đến một số dần nhỏ đi cho đến khi kết quả được làm tròn thành không.
Ta đã thảo luận vấn đề này dưới dạng vấn đề lý thuyết trong: numref: `sec_maximum_likelkel`, nhưng ta thấy hiện tượng này rõ ràng ở đây trong thực tế.


<!--
As discussed in that section, we fix this by use the fact that $\log a b = \log a + \log b$, i.e., we switch to summing logarithms.
Even if both $a$ and $b$ are small numbers, the logarithm values should be in a proper range.
-->

Như đã thảo luận trong phần đó, ta khắc phục điều này bằng cách sử dụng tính chất $\log a b = \log a + \log b$, cụ thể là ta chuyển sang tính tổng các logarit.
Nhờ vậy ngay cả khi cả $a$ và $b$ đều là các số nhỏ, giá trị các logarit sẽ nằm trong miền thích hợp.


```{.python .input}
a = 0.1
print('underflow:', a**784)
print('logarithm is normal:', 784*math.log(a))
```

```{.python .input}
#@tab pytorch
a = 0.1
print('underflow:', a**784)
print('logarithm is normal:', 784*math.log(a))
```

```{.python .input}
#@tab tensorflow
a = 0.1
print('underflow:', a**784)
print('logarithm is normal:', 784*tf.math.log(a).numpy())
```


<!--
Since the logarithm is an increasing function, we can rewrite :eqref:`eq_naive_bayes_estimation` as
-->

Vì logarit là một hàm tăng dần, ta có thể viết lại :eqref: `eq_naive_bayes_estimation` thành


$$ \hat{y} = \mathrm{argmax}_y \> \sum_{i=1}^d \log P_{xy}[x_i, y] + \log P_y[y].$$


<!--
We can implement the following stable version:
-->

Ta có thể lập trình phiên bản ổn định sau:


```{.python .input}
log_P_xy = np.log(P_xy)
log_P_xy_neg = np.log(1 - P_xy)
log_P_y = np.log(P_y)

def bayes_pred_stable(x):
    x = np.expand_dims(x, axis=0)  # (28, 28) -> (1, 28, 28)
    p_xy = log_P_xy * x + log_P_xy_neg * (1 - x)
    p_xy = p_xy.reshape(10, -1).sum(axis=1)  # p(x|y)
    return p_xy + log_P_y

py = bayes_pred_stable(image)
py
```

```{.python .input}
#@tab pytorch
log_P_xy = torch.log(P_xy)
log_P_xy_neg = torch.log(1 - P_xy)
log_P_y = torch.log(P_y)

def bayes_pred_stable(x):
    x = x.unsqueeze(0)  # (28, 28) -> (1, 28, 28)
    p_xy = log_P_xy * x + log_P_xy_neg * (1 - x)
    p_xy = p_xy.reshape(10, -1).sum(axis=1)  # p(x|y)
    return p_xy + log_P_y

py = bayes_pred_stable(image)
py
```

```{.python .input}
#@tab tensorflow
log_P_xy = tf.math.log(P_xy)
# TODO: Look into why this returns infs
log_P_xy_neg = tf.math.log(1 - P_xy)
log_P_y = tf.math.log(P_y)

def bayes_pred_stable(x):
    x = tf.expand_dims(x, axis=0)  # (28, 28) -> (1, 28, 28)
    p_xy = log_P_xy * x + log_P_xy_neg * (1 - x)
    p_xy = tf.math.reduce_sum(tf.reshape(p_xy, (10, -1)), axis=1)  # p(x|y)
    return p_xy + log_P_y

py = bayes_pred_stable(image)
py
```


<!--
We may now check if the prediction is correct.
-->

Bây giờ ta có thể kiểm tra liệu dự đoán này có đúng hay không.

<!-- ===================== Kết thúc dịch Phần 3 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 4 ===================== -->

```{.python .input}
# Convert label which is a scalar tensor of int32 dtype
# to a Python scalar integer for comparison
py.argmax(axis=0) == int(label)
```

```{.python .input}
#@tab pytorch
py.argmax(dim=0) == label
```

```{.python .input}
#@tab tensorflow
tf.argmax(py, axis=0) == label
```


<!--
If we now predict a few validation examples, we can see the Bayes
classifier works pretty well.
-->

Nếu ta dự đoán một vài mẫu kiểm định, ta có thể thấy 
bộ phân loại Bayes hoạt động khá tốt.


```{.python .input}
def predict(X):
    return [bayes_pred_stable(x).argmax(axis=0).astype(np.int32) for x in X]

X, y = mnist_test[:18]
preds = predict(X)
d2l.show_images(X, 2, 9, titles=[str(d) for d in preds]);
```

```{.python .input}
#@tab pytorch
def predict(X):
    return [bayes_pred_stable(x).argmax(dim=0).type(torch.int32).item() 
            for x in X]

X = torch.stack([mnist_train[i][0] for i in range(10,38)], dim=1).squeeze(0)
y = torch.tensor([mnist_train[i][1] for i in range(10,38)])
preds = predict(X)
d2l.show_images(X, 2, 9, titles=[str(d) for d in preds]);
```

```{.python .input}
#@tab tensorflow
def predict(X):
    return [tf.cast(tf.argmax(bayes_pred_stable(x), axis=0), tf.int32).numpy()
            for x in X]

X = tf.stack([tf.cast(train_images[i], tf.float32) for i in range(10, 38)], axis=0)
y = tf.constant([train_labels[i] for i in range(10, 38)])
preds = predict(X)
# TODO: The preds are not correct due to issues with bayes_pred_stable()
d2l.show_images(X, 2, 9, titles=[str(d) for d in preds]);
```


<!--
Finally, let us compute the overall accuracy of the classifier.
-->

Cuối cùng, chúng ta hãy tính toán độ chính xác tổng thể của bộ phân loại.


```{.python .input}
X, y = mnist_test[:]
preds = np.array(predict(X), dtype=np.int32)
float((preds == y).sum()) / len(y)  # Validation accuracy
```

```{.python .input}
#@tab pytorch
X = torch.stack([mnist_train[i][0] for i in range(len(mnist_test))], 
                dim=1).squeeze(0)
y = torch.tensor([mnist_train[i][1] for i in range(len(mnist_test))])
preds = torch.tensor(predict(X), dtype=torch.int32)
float((preds == y).sum()) / len(y)  # Validation accuracy
```

```{.python .input}
#@tab tensorflow
X = tf.stack([tf.cast(train_images[i], tf.float32) for i in range(len(test_images))], axis=0)
y = tf.constant([train_labels[i] for i in range(len(test_images))])
preds = tf.constant(predict(X), dtype=tf.int32)
# TODO: The accuracy is not correct due to issues with bayes_pred_stable()
tf.reduce_sum(tf.cast(preds == y, tf.float32)) / len(y)  # Validation accuracy
```


<!--
Modern deep networks achieve error rates of less than $0.01$.
The relatively poor performance is due to the incorrect statistical assumptions that we made in our model: 
we assumed that each and every pixel are *independently* generated, depending only on the label.
This is clearly not how humans write digits, and this wrong assumption led to the downfall of our overly naive (Bayes) classifier.
-->

Các mạng sâu hiện đại đạt được tỷ lệ lỗi dưới $0,01$.
Hiệu suất tương đối kém là do các giả định thống kê không chính xác mà chúng ta đã đưa vào trong mô hình của mình:
ta đã giả định rằng mỗi và mọi pixel được tạo *một cách độc lập*, chỉ phụ thuộc vào nhãn.
Đây rõ ràng không phải là cách con người viết các chữ số, và giả định sai lầm này đã dẫn đến sự thua kém của bộ phân loại quá ngây thơ (*naive* Bayes) của chúng ta.


## Tóm tắt

<!--
* Using Bayes' rule, a classifier can be made by assuming all observed features are independent.  
* This classifier can be trained on a dataset by counting the number of occurrences of combinations of labels and pixel values.
* This classifier was the gold standard for decades for tasks such as spam detection.
-->

* Sử dụng quy tắc Bayes, một bộ phân loại có thể được tạo ra bằng cách giả định tất cả các đặc trưng quan sát được là độc lập.
* Bộ phân loại này có thể được huấn luyện trên tập dữ liệu bằng cách đếm số lần xuất hiện của các tổ hợp nhãn và giá trị pixel.
* Bộ phân loại này là tiêu chuẩn vàng trong nhiều thập kỷ cho các tác vụ như phát hiện thư rác.


## Bài tập

<!--
1. Consider the dataset $[[0,0], [0,1], [1,0], [1,1]]$ with labels given by the XOR of the two elements $[0,1,1,0]$.
What are the probabilities for a Naive Bayes classifier built on this dataset.
Does it successfully classify our points? If not, what assumptions are violated?
2. Suppose that we did not use Laplace smoothing when estimating probabilities and a data example arrived at testing time which contained a value never observed in training.
What would the model output?
3. The naive Bayes classifier is a specific example of a Bayesian network, where the dependence of random variables are encoded with a graph structure.
While the full theory is beyond the scope of this section (see :cite:`Koller.Friedman.2009` for full details), 
explain why allowing explicit dependence between the two input variables in the XOR model allows for the creation of a successful classifier.
-->

1. Xem xét tập dữ liệu $[[0,0], [0,1], [1,0], [1,1]]$ với các nhãn được cung cấp bởi phép XOR của hai phần tử $[0,1,1,0]$.
Các xác suất cho bộ phân loại Naive Bayes được xây dựng trên tập dữ liệu này là bao nhiêu?
Nó có phân loại thành công các điểm dữ liệu không? Nếu không, những giả định nào bị vi phạm?
2. Giả sử rằng ta không sử dụng phép làm mượt Laplace khi ước tính xác suất và có một mẫu dữ liệu tại thời điểm kiểm tra chứa một giá trị chưa bao giờ được quan sát trong quá trình huấn luyện.
Lúc này mô hình sẽ trả về giá trị gì?
3. Bộ phân loại Naive Bayes là một ví dụ cụ thể của mạng Bayes, trong đó sự phụ thuộc của các biến ngẫu nhiên được mã hóa bằng cấu trúc đồ thị.
Mặc dù lý thuyết đầy đủ nằm ngoài phạm vi của phần này (xem :cite:`Koller.Friedman.2009` để biết đầy đủ chi tiết),
hãy giải thích tại sao việc đưa sự phụ thuộc tường minh giữa hai biến đầu vào trong mô hình XOR lại có thể tạo ra một bộ phân loại thành công.



<!-- ===================== Kết thúc dịch Phần 4 ===================== -->
<!-- ========================================= REVISE - KẾT THÚC ===================================-->


## Thảo luận
* Tiếng Anh: [MXNet](https://discuss.d2l.ai/t/418)
* Tiếng Việt: [Diễn đàn Machine Learning Cơ Bản](https://forum.machinelearningcoban.com/c/d2l)


## Những người thực hiện
Bản dịch trong trang này được thực hiện bởi:
<!--
Tác giả của mỗi Pull Request điền tên mình và tên những người review mà bạn thấy
hữu ích vào từng phần tương ứng. Mỗi dòng một tên, bắt đầu bằng dấu `*`.

Tên đầy đủ của các reviewer có thể được tìm thấy tại https://github.com/aivivn/d2l-vn/blob/master/docs/contributors_info.md
-->

* Đoàn Võ Duy Thanh
<!-- Phần 1 -->
* Trần Yến Thy
* Lê Khắc Hồng Phúc
* Phạm Minh Đức

<!-- Phần 2 -->
* Trần Yến Thy

<!-- Phần 3 -->
* Nguyễn Mai Hoàng Long

<!-- Phần 4 -->
* Trần Yến Thy
* Lê Khắc Hồng Phúc
* Phạm Minh Đức

*Lần cập nhật gần nhất: 10/09/2020. (Cập nhật lần cuối từ nội dung gốc: 05/08/2020)*
