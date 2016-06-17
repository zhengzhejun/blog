---
layout: post
title: Spark MLlib之Random Forest使用与实现细节
comments: true
categories: [Spark, 机器学习]
tags: [MLlib, 随机森林]
---

最近在公司有个需求，就是在通过HDFS上数据，利用Spark的MLlib训练一个回归模型，用于线上排序。一开始训练了LR，但是效果不佳。分析了以后，发现很多特征与label之间并没有明显的线性关系，因此，LR效果不佳。所以，我们准备使用RandomForest来试一下。在测试集中的表现良好，AUC值提高了7%。但是，训练好的RandomForest模型用于线上排序的时候，遇到了一个问题。就是MLlib对于每一个model都提供了model.save和model.load的方法，但是线上环境由于一些原因导致无法添加spark-core和spark-mllib的dependency。所以，这里我需要自己实现模型的存储和解析的方法。所以我就趁这个机会好好学习一下MLlib中RandomForest的实现方法。

## RandomForest Example
下面是官方文档中的样例程序。随机森林即可以用来做分类，也可以用来做回归。
{% highlight scala %}
import org.apache.spark.mllib.tree.RandomForest
import org.apache.spark.mllib.tree.model.RandomForestModel
import org.apache.spark.mllib.util.MLUtils

val data = MLUtils.loadLibSVMFile(sc, "data/mllib/sample_libsvm_data.txt")
// Split the data into training and test sets (30% held out for testing)
val splits = data.randomSplit(Array(0.7, 0.3))
val (trainingData, testData) = (splits(0), splits(1))

val numClasses = 2
val categoricalFeaturesInfo = Map[Int, Int]()
val numTrees = 3 // Use more in practice.
val featureSubsetStrategy = "auto" // Let the algorithm choose.
val impurity = "gini"
val maxDepth = 4
val maxBins = 32

val model = RandomForest.trainClassifier(trainingData, numClasses, categoricalFeaturesInfo,
  numTrees, featureSubsetStrategy, impurity, maxDepth, maxBins)

val labelAndPreds = testData.map { point =>
  val prediction = model.predict(point.features)
  (point.label, prediction)
}
val testErr = labelAndPreds.filter(r => r._1 != r._2).count.toDouble / testData.count()
println("Test Error = " + testErr)
println("Learned classification forest model:\n" + model.toDebugString)

// Save and load model
model.save(sc, "target/tmp/myRandomForestClassificationModel")
val sameModel = RandomForestModel.load(sc, "target/tmp/myRandomForestClassificationModel")
{% endhighlight %}
