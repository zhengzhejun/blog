---
layout: post
title: TensorFlow之TFRecord和Sparse tensor的使用
comments: true
categories: [TensorFlow, DeepLearning]
tags: [TFRecord, SparseTensor]
---

## TFRecord Example
首先我们需要将原始数据转换成TFRecord格式。首先我准备了样例数据，该数据格式为libsvm格式。
{% highlight vim %}
0 5:1 6:1 17:1 21:1 35:1 40:1 53:1 63:1 71:1 73:1 74:1 76:1 80:1 83:1
1 5:1 7:1 17:1 22:1 36:1 40:1 51:1 63:1 73:1 74:1 76:1 81:1 83:1
1 2:1 6:1 14:1 29:1 39:1 42:1 52:1 64:1 67:1 72:1 75:1 76:1 82:1 83:1
1 4:1 6:1 16:1 63:1 67:1 73:1 75:1 76:1 80:1 83:1
{% endhighlight %}
将上述数据转换成TFRecord格式,并从TFRecord格式中读取数据。代码如下：
{% highlight python %}
def convert_tfrecords(input_filename, output_filename):
    if os.path.exists(output_filename):
        os.remove(output_filename)
    current_path = os.getcwd()
    input_file = os.path.join(current_path, input_filename)
    output_file = os.path.join(current_path, output_filename)
    print("Start to convert {} to {}".format(input_file, output_file))

    writer = tf.python_io.TFRecordWriter(output_file)

    #这里我们的数据表示方法为稀疏表示方法，我们需要将特征Id，以及特征Id对应的特征值分别记录下来
    for line in open(input_file, "r"):
        data = line.split(" ")
        label = float(data[0])
        ids = [] # id表示特征Id
        values = [] # values表示特征Id对应的值
        for fea in data[1:]:
            id, value = fea.split(":")
            ids.append(int(id))
            values.append(float(value))

        # Write each example one by one
        example = tf.train.Example(features=tf.train.Features(feature={
            "label":
                tf.train.Feature(float_list=tf.train.FloatList(value=[label])),
            "ids":
                tf.train.Feature(int64_list=tf.train.Int64List(value=ids)),
            "values":
                tf.train.Feature(float_list=tf.train.FloatList(value=values))
        }))

        writer.write(example.SerializeToString())

    writer.close()
    print("Successfully convert {} to {}".format(input_file, output_file))

def read_and_decode(filename_queue):
    reader = tf.TFRecordReader()
    _, serialized_example = reader.read(filename_queue)
    # 这里，我们在取label字段的时候使用tf.FixedLenFeature，而在读取ids，values字段的时候，使用VarLenFeature。
    """
    This op parses serialized examples into a dictionary mapping keys to `Tensor`
    and `SparseTensor` objects. `features` is a dict from keys to `VarLenFeature`
    and `FixedLenFeature` objects. Each `VarLenFeature` is mapped to a
    `SparseTensor`, and each `FixedLenFeature` is mapped to a `Tensor`.
    """
    # 根据parse_example的文档，我们可以看到，FixedLenFeature对应Tensor，VarLenFeature对应Sparse Tensor。
    # 这里之所以用到VarLenfeature，是因为我们的原数据为稀疏表达形式，每行feature个数不同，所以需要借助VarLenFeature。
    features = tf.parse_single_example(serialized_example,
                                       features = {
                                           "label" : tf.FixedLenFeature([], tf.float32),
                                           "ids" : tf.VarLenFeature(tf.int64),
                                           "values" : tf.VarLenFeature(tf.float32)
                                       })
    label = features["label"]
    ids = features["ids"]
    values = features["values"]
    return label, ids, values

current_path = os.getcwd()
for file in os.listdir(current_path):
    if file.endswith(".libsvm"):
        convert_tfrecords(file, file + ".tfrecords")
filename = os.path.join(current_path, "data.libsvm.tfrecords")
filename_queue = tf.train.string_input_produces([filename], num_epochs=1)
label, ids, values = read_and_decode(filename_queue)
{% endhighlight %}

上面的代码是用来读取每行feature个数不相同的数据。当原数据中每行feature个数相同，则可以通过下面的方式读取。

{% highlight python %}
"""
这里的数据格式为：
10,10,10,8,6,1,8,9,1,1
6,2,1,1,1,1,7,1,1,0
2,5,3,3,6,7,7,5,1,1
最后一列为label，前面都是feature，并且每行的feature个数相同
"""
def convert_tfrecords(input_filename, output_filename):
    current_path = os.getcwd()
    input_file = os.path.join(current_path, input_filename)
    output_file = os.path.join(current_path, output_filename)
    print("Start to convert {} to {}".format(input_file, output_file))

    writer = tf.python_io.TFRecordWriter(output_file)

    for line in open(input_file, "r"):
        # Split content in CSV file
        data = line.split(",")
        label = float(data[9])
        features = [float(i) for i in data[0:9]]

        # Write each example one by one
        example = tf.train.Example(features=tf.train.Features(feature={
            "label":
                tf.train.Feature(float_list=tf.train.FloatList(value=[label])),
            "features":
                tf.train.Feature(float_list=tf.train.FloatList(value=features)),
        }))

        writer.write(example.SerializeToString())

    writer.close()
    print("Successfully convert {} to {}".format(input_file, output_file))
def read_and_decode(filename_queue):
  reader = tf.TFRecordReader()
  _, serialized_example = reader.read(filename_queue)
  features = tf.parse_single_example(
      serialized_example,
      features={
          "label": tf.FixedLenFeature([], tf.float32),
          "features": tf.FixedLenFeature([FEATURE_SIZE], tf.float32),
      })
  label = features["label"]
  features = features["features"]
  return label, features
{% endhighlight %}

总结起来，使用TFRecord格式，我们需要以下几个函数和步骤

### 总结
* 首先初始化TFRecordWriter，tf.python_io.TFRecordWriter(output_file)
* 然后读取原数据，生成example = tf.train.Example()，并写入文件，writer.write(example.SerializeToString())
* 在读取阶段，首先通过tf.train.string_input_producer([filename], num_epochs=1)来得到filename_queue
* 然后初始化reader = tf.TFRecordReader(), 读取数据，_, serialized_example = reader.read(filename_queue)得到反序列化的数据

这里我们注意到，我们使用tf.parse_single_example来进行反序列化，所以，我们得到的数据都是一维数据，即得到的sparse tensor的shape=array([14])，这也意味着sparse tensor的indices和values都是一维向量。因此，我们想要根据ids和values来初始化sparse tensor的话，我们需要做一些转变。
SparseTensor的初始化参数定义如下：
{% highlight python %}
"""
      indices: A 2-D int64 tensor of shape `[N, ndims]`.
      values: A 1-D tensor of any type and shape `[N]`.
      shape: A 1-D int64 tensor of shape `[ndims]`.
      SparseTensor(indices=[[0, 0], [1, 2]], values=[1, 2], shape=[3, 4])
"""
{% endhighlight %}
我们可以看到，indices是二维向量，values则为一维向量，所以我们需要把我们从TFRecord得到的ids和values转换成相应的格式。

{% highlight python %}
"""
这里我们传入上一步得到的sparse_id和sparse_values。这两个参数都是SparseTensor格式。weight是一个自定义的变量，用于做乘法。

"""
def sparse_tensor_test(sparse_id, sparse_values, weights):
    #这里我们将sparse_id从SparseTensor转换成Tensor，shape依然是一维的。
    dense_ids = tf.sparse_tensor_to_dense(sparse_id, 0)
    #初始化zero
    zero = tf.zeros(tf.shape(dense_ids), dtype=tf.int64)
    #将zero和dense_ids合并
    final_dense_id = tf.pack([zero, dense_ids], axis=1)
    dense_value = tf.sparse_tensor_to_dense(sparse_values, 0)

    sp_tensor = tf.SparseTensor(final_dense_id, dense_value, shape=[1, 84])
    mul_result = tf.sparse_tensor_dense_matmul(sp_tensor, weights)
    return mul_result

{% endhighlight %}

### 按batch来读
按照batch来读的话，与上面每次读一行数据，有所区别。主要是增加了tf.train.shuffle_batch函数。并且有以下两种方式都可以读batch。
{% highlight python %}
def read_and_decode_batch(filename_queue):
    reader = tf.TFRecordReader()
    _, serialized_example = reader.read(filename_queue)
    features = tf.parse_single_example(serialized_example,
                                       features={
                                           "label": tf.FixedLenFeature([], tf.float32),
                                           "ids": tf.VarLenFeature(tf.int64),
                                           "values": tf.VarLenFeature(tf.float32)
                                       })
    label = features["label"]
    ids = features["ids"]
    values = features["values"]

    batch_label, batch_ids, batch_values = tf.train.shuffle_batch([label, ids, values], batch_size=1, capacity=5,
                                                                  min_after_dequeue=3)
    return batch_label, batch_ids, batch_values

def read_and_decode_batch2(filename_queue):
    reader = tf.TFRecordReader()
    _, serialized_example = reader.read(filename_queue)
    batch_serialized_example = tf.train.shuffle_batch([serialized_example], batch_size=1, capacity=5, min_after_dequeue=3)
    features = tf.parse_example(batch_serialized_example,
                                       features={
                                           "label": tf.FixedLenFeature([], tf.float32),
                                           "ids": tf.VarLenFeature(tf.int64),
                                           "values": tf.VarLenFeature(tf.float32)
                                       })
    batch_label = features["label"]
    batch_ids = features["ids"]
    batch_values = features["values"]

    return batch_label, batch_ids, batch_values

"""
对于batch数据，parse_example返回的数据为二维数据，所以，我们需要对数据做一下flatten
"""
def sparse_tensor_test_batch(sparse_ids, sparse_values, weights):
    dense_values = tf.sparse_tensor_to_dense(sparse_values, 0)
    #对于dense_values做flatten
    dense_values_flatten = tf.reshape(dense_values, [-1])

    dense_ids = tf.sparse_tensor_to_dense(sparse_ids, 0)
    dense_ids_shape = tf.shape(dense_ids)
    zero = tf.zeros(dense_ids_shape, dtype=tf.int64)

    #对于dense_values做flatten
    flatten_dense_ids = tf.reshape(dense_ids, [-1])
    flatten_zero = tf.reshape(zero, [-1])

    dense_ids_final = tf.pack([flatten_zero, flatten_dense_ids], axis=1)
    sp_tensor = tf.SparseTensor(dense_ids_final, dense_values_flatten, [1, 84])
    mul_result = tf.sparse_tensor_dense_matmul(sp_tensor, weights)
    return mul_result
{% endhighlight %}


## 全部代码
{% highlight python %}
import tensorflow as tf
import os
import numpy as np


def convert_tfrecords(input_filename, output_filename):
    if os.path.exists(output_filename):
        os.remove(output_filename)
    current_path = os.getcwd()
    input_file = os.path.join(current_path, input_filename)
    output_file = os.path.join(current_path, output_filename)
    print("Start to convert {} to {}".format(input_file, output_file))

    writer = tf.python_io.TFRecordWriter(output_file)

    for line in open(input_file, "r"):
        data = line.split(" ")
        label = float(data[0])
        ids = []
        values = []
        for fea in data[1:]:
            id, value = fea.split(":")
            ids.append(int(id))
            values.append(float(value))

        # Write each example one by one
        example = tf.train.Example(features=tf.train.Features(feature={
            "label":
                tf.train.Feature(float_list=tf.train.FloatList(value=[label])),
            "ids":
                tf.train.Feature(int64_list=tf.train.Int64List(value=ids)),
            "values":
                tf.train.Feature(float_list=tf.train.FloatList(value=values))
        }))

        writer.write(example.SerializeToString())

    writer.close()
    print("Successfully convert {} to {}".format(input_file, output_file))


def read_and_decode(filename_queue):
    reader = tf.TFRecordReader()
    _, serialized_example = reader.read(filename_queue)
    features = tf.parse_single_example(serialized_example,
                                       features={
                                           "label": tf.FixedLenFeature([], tf.float32),
                                           "ids": tf.VarLenFeature(tf.int64),
                                           "values": tf.VarLenFeature(tf.float32)
                                       })
    label = features["label"]
    ids = features["ids"]
    values = features["values"]
    return label, ids, values


current_path = os.getcwd()
for file in os.listdir(current_path):
    if file.endswith(".libsvm"):
        convert_tfrecords(file, file + ".tfrecords")


def read_and_decode_batch(filename_queue):
    reader = tf.TFRecordReader()
    _, serialized_example = reader.read(filename_queue)
    features = tf.parse_single_example(serialized_example,
                                       features={
                                           "label": tf.FixedLenFeature([], tf.float32),
                                           "ids": tf.VarLenFeature(tf.int64),
                                           "values": tf.VarLenFeature(tf.float32)
                                       })
    label = features["label"]
    ids = features["ids"]
    values = features["values"]

    batch_label, batch_ids, batch_values = tf.train.shuffle_batch([label, ids, values], batch_size=1, capacity=5,
                                                                  min_after_dequeue=3)
    return batch_label, batch_ids, batch_values

def read_and_decode_batch2(filename_queue):
    reader = tf.TFRecordReader()
    _, serialized_example = reader.read(filename_queue)
    batch_serialized_example = tf.train.shuffle_batch([serialized_example], batch_size=1, capacity=5, min_after_dequeue=3)
    features = tf.parse_example(batch_serialized_example,
                                       features={
                                           "label": tf.FixedLenFeature([], tf.float32),
                                           "ids": tf.VarLenFeature(tf.int64),
                                           "values": tf.VarLenFeature(tf.float32)
                                       })
    batch_label = features["label"]
    batch_ids = features["ids"]
    batch_values = features["values"]

    return batch_label, batch_ids, batch_values

def embedding_lookup_test(sparse_ids, sparse_values, weights):
    tf.nn.embedding_lookup_sparse(weights, sparse_ids, sparse_values, combiner="sum")


def sparse_tensor_test_batch(sparse_ids, sparse_values, weights):
    dense_values = tf.sparse_tensor_to_dense(sparse_values, 0)
    dense_values_flatten = tf.reshape(dense_values, [-1])

    dense_ids = tf.sparse_tensor_to_dense(sparse_ids, 0)
    dense_ids_shape = tf.shape(dense_ids)
    zero = tf.zeros(dense_ids_shape, dtype=tf.int64)

    flatten_dense_ids = tf.reshape(dense_ids, [-1])
    flatten_zero = tf.reshape(zero, [-1])

    dense_ids_final = tf.pack([flatten_zero, flatten_dense_ids], axis=1)
    sp_tensor = tf.SparseTensor(dense_ids_final, dense_values_flatten, [1, 84])
    mul_result = tf.sparse_tensor_dense_matmul(sp_tensor, weights)
    return mul_result


def sparse_tensor_test(sparse_id, sparse_values, weights):
    dense_ids = tf.sparse_tensor_to_dense(sparse_id, 0)
    zero = tf.zeros(tf.shape(dense_ids), dtype=tf.int64)
    final_dense_id = tf.pack([zero, dense_ids], axis=1)
    dense_value = tf.sparse_tensor_to_dense(sparse_values, 0)

    sp_tensor = tf.SparseTensor(final_dense_id, dense_value, shape=[1, 84])
    mul_result = tf.sparse_tensor_dense_matmul(sp_tensor, weights)
    return mul_result


filename = os.path.join(current_path, "data.libsvm.tfrecords")
filename_queue = tf.train.string_input_producer([filename], num_epochs=1)
# label, ids, values = read_and_decode(filename_queue)
label_batch, ids_batch, values_batch = read_and_decode_batch2(filename_queue)
dense_ids = tf.sparse_tensor_to_dense(ids_batch, 0)

weights_arr = np.ones(shape=[84, 1], dtype=np.float32)
# weights = tf.get_variable("weights", [84, 1], initializer=tf.random_normal_initializer())
weights = tf.Variable(weights_arr)
# mul_result = sparse_tensor_test(ids_batch, values_batch, weights)
mul_result_batch = sparse_tensor_test_batch(ids_batch, values_batch, weights)

with tf.Session() as sess:
    init = tf.initialize_local_variables()
    global_init = tf.initialize_all_variables()
    sess.run(init)
    sess.run(global_init)
    coord = tf.train.Coordinator()
    threads = tf.train.start_queue_runners(coord=coord, sess=sess)

    try:
        while not coord.should_stop():
            # result_label, result_ids, result_values, mul_result_value = sess.run([label, ids, values, mul_result])
            mul_result_value = sess.run([dense_ids])

            # print result_label, result_ids, result_values, mul_result_value
            # print result_label, result_ids, dense_ids_values, result_values
            print mul_result_value
    except tf.errors.OutOfRangeError:
        print("Done training after reading all data")
    finally:
        coord.request_stop()
    coord.join(threads)
{% endhighlight %}

