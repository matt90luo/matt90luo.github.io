---
title: pysparky工程建立及tensorflow-on-spark
tags:
---
通过anaconda建立一个python环境并且使这个环境应用在spark on yarn 上

1. 安装anaconda  
    - wget -c https://repo.anaconda.com/archive/Anaconda3-2019.07-Linux-x86_64.sh
    - bash ./Anaconda3-2019.07-Linux-x86_64.sh 安装anaconda 注意anaconda 根目录设置
2. conda 使用
    - conda config --set auto_activate_base false 这样设置anaconda不会自动激活
    - conda config --add channels conda-forge  添加conda channel
    - conda list -n XXX  查看指定环境的安装包
    - conda search XXX  查找package信息
    - conda undate -n evn package  更新指定环境的包
    - conda deactivate 退出当前环境
    - conda activate XXX 激活某个环境
    - conda install -n pyeda_env tensorflow-datasets 指定环境安装指定包
3. conda create --name pyeda_env --quiet  --copy --yes python=3.6.9 numpy tensorflow pyspark tensorflowonspark 创建环境

4. zip -r -9 -q ~/manley/pyeda_env.zip ./pyeda_env/  将刚刚创建的环境整个打包压缩
5. 将zip文件放到hdfs上 我的地址是 /tmp/manley/pyeda_env.zip
6. 提交命令 spark-submit --master yarn --num-executors 16 --executor-cores 2 --executor-memory 4G  --driver-memory 16G --conf "spark.driver.cores=10" --archives hdfs:///tmp/manley/pyeda_env.zip#Python --conf spark.pyspark.python=./Python/pyeda_env/bin/python --conf spark.pyspark.driver.python=./anaconda3/envs/pyeda_env/bin/python foo_foo_tf.py

对于提交命令的几点说明
* spark-submit 默认是client模式, master是跑在提交机器上, 所以需要分别配置 spark.pyspark.python 和 spark.pyspark.driver.python
* --archives 每个executor会把这个参数指定的所有文件下载到working directory 我们也是利用这个特性将这个python环境以及依赖包传到每个executor, 后面的#符号相当于连接了一个重命名, 所以executor 的路径可以是 \./<重命名>

# TIPS
- Currently, PySpark can not support pickle a class object in current script ( 'main'), the workaround could be put the implementation of the class into a separate module, then use "bin/spark-submit --py-files xxx.py" in deploy it. 简单来说 结构体单独定义成模块, 然后使用命令--py-files 进行分发

# 利用tensorflow计算矩阵相乘
背景是一台单卡主机作为pyspark提交主机(client模式), tensorflow版本为2.0, 但是实际中以v1.0的兼容模式运行, pypark进行常规数据处理, 利用pyspark提供的toLocalIterator() 接口实现按照分区将数据fetch回driver并进行训练
在driver memory 的分配上 建议不小于GPU可用内存, 这样可以实现并行计算的最大化, 核心代码如下:
```python
    def step_f(sess, user_iter, name):
        mat1 = tf.compat.v1.placeholder(tf.float16, shape=(None, dim))
        mat2 = tf.compat.v1.placeholder(tf.float16, shape=(None, dim))
        output = tf.compat.v1.matmul(mat1, mat2, transpose_b=True)
        topk_index = tf.compat.v1.nn.top_k(output, 500).indices #build the compute graph

        feed = {mat1: np.float16(user_iter[1]), mat2: items}
        tmp = sess.run(topk_index, feed_dict=feed)
        res = [user_iter[0][i] + "," +":".join(item_arr[tmp[i]]) for i in range(len(tmp))]
        sc.parallelize(res).saveAsTextFile("hdfs:///tmp/manley/tffoo/" + name)

    with tf.compat.v1.Session() as sess:
        i = 0
        for u in tqdm(users.toLocalIterator()):
            step_f(sess,u, str(i))
            i += 1

    users.unpersist()
    print("res count ", sc.textFile("/tmp/manley/tffoo/*").count())

```
同时注意上面的例子会将结果作为RDD写会需要保证有足够的内存来处理, 推荐将RDD先保存随后再使用


# py4j使用
Py4J enables Python programs running in a Python interpreter to dynamically access Java objects in a Java Virtual Machine. Methods are called as if the Java objects resided in the Python interpreter and Java collections can be accessed through standard Python collection methods. Py4J also enables Java programs to call back Python objects.
Py4j可以使运行于python解释器的python程序动态的访问java虚拟机中的java对象。Java方法可以像java对象就在python解释器里一样被调用，Java collection也可以通过标准python collection方法调用。Py4j也可以使java程序回调python对象

python如何和java 的JVM通信?最简单的就是RPC. JVM 作为RPC的服务端， python app 作为RPC 的客户端. JVM 会开启一个Socket 端口提供服务， python app 只需要调用py4j 提供的client 的接口即可. (需要指出py4j 并不会启动一个JVM， 需要java程序)

20/01/11 21:59:06 INFO StateStoreCoordinatorRef: Registered StateStoreCoordinator endpoint
npartition num  478
0it [00:00, ?it/s]
(348, 617548)
1it [12:23, 743.62s/i

