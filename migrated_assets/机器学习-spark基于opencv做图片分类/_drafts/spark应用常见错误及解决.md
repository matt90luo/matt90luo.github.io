---
title: spark应用常见错误及解决
tags:
---

## spark编程模型介绍
1. 弹性式分布式数据集RDD 抽象对象 Resilient Distributed Dataset
2. 广播变量和累加器

- 分区
- 函数
- 依赖
- 优先位置
- 分区策略 根据分区模式和数据存放的位置 k-v对RDD根据哈希值分区 类似于partioner接口 

对于广播变量，尽量避免使用容器嵌套的形式，非常容易导致内存异常，可以先转成字符串，再进行传输
WARN Utils: Truncated the string representation of a plan since it was too large. This behavior can be adjusted by setting 'spark.debug.maxToStringFields' in SparkEnv.conf.




案列一
RDD结构RDD[cookieid, list[obj]] 大小为百万记录, 和另一个RDD[cookieid]需要做关联, 决定采用
广播RDD[cookieid]的方式 过程中分别出现以下问题
1. spark.kryoserializer.buffer.max 设置过小, 广播时需要的buffer 根据实际系列化的对象大小 适当调大
2. Total size of serialized results of 1 tasks (1499.7 MB) is bigger than spark.driver.maxResultSize (1024.0 MB)
3. 由于spark内部采用的kryo版本存在bug 在序列化的列表对象长度超过10亿时会产生 serializing large (> 1 billion) objects may result in a java.lang.NegativeArraySizeException
解决办法是 临时设置 spark.kryo.refferenceTrackingEnabled=false


## spark 矩阵计算
1. Local vector
A local vector has integer-typed and 0-based indices and double-typed values, stored on a single machine. MLlib supports two types of local vectors: dense and sparse. A dense vector is backed by a double array representing its entry values, while a sparse vector is backed by two parallel arrays: indices and values. For example, a vector (1.0, 0.0, 3.0) can be represented in dense format as [1.0, 0.0, 3.0] or in sparse format as (3, [0, 2], [1.0, 3.0]), where 3 is the size of the vector.

2. Labeled point
A labeled point is a local vector, either dense or sparse, associated with a label/response. In MLlib, labeled points are used in supervised learning algorithms. We use a double to store a label, so we can use labeled points in both regression and classification. For binary classification, a label should be either 0 (negative) or 1 (positive). For multiclass classification, labels should be class indices starting from zero: 0, 1, 2, ....

3. Sparse data
It is very common in practice to have sparse training data. MLlib supports reading training examples stored in LIBSVM format, which is the default format used by LIBSVM and LIBLINEAR. It is a text format in which each line represents a labeled sparse feature vector using the following format:

>label index1:value1 index2:value2 ...

4. laocal matrix
A local matrix has integer-typed row and column indices and double-typed values, stored on a single machine. MLlib supports dense matrices, whose entry values are stored in a single double array in column-major order, and sparse matrices, whose non-zero entry values are stored in the Compressed Sparse Column (CSC) format in column-major order. For example, the following dense matrix
The base class of local matrices is Matrix, and we provide two implementations: DenseMatrix, and SparseMatrix. We recommend using the factory methods implemented in Matrices to create local matrices. Remember, local matrices in MLlib are stored in column-major order.

``` scala
import org.apache.spark.mllib.linalg.{Matrix, Matrices}
// Create a dense matrix ((1.0, 2.0), (3.0, 4.0), (5.0, 6.0))
val dm: Matrix = Matrices.dense(3, 2, Array(1.0, 3.0, 5.0, 2.0, 4.0, 6.0))
// Create a sparse matrix ((9.0, 0.0), (0.0, 8.0), (0.0, 6.0))
val sm: Matrix = Matrices.sparse(3, 2, Array(0, 1, 3), Array(0, 2, 1), Array(9, 6, 8))
```

5. Distributed matrix
A distributed matrix has long-typed row and column indices and double-typed values, stored distributively in one or more RDDs. It is very important to choose the right format to store large and distributed matrices. Converting a distributed matrix to a different format may require a global shuffle, which is quite expensive. Four types of distributed matrices have been implemented so far.
The basic type is called RowMatrix. A RowMatrix is a row-oriented distributed matrix without meaningful row indices, e.g., a collection of feature vectors. It is backed by an RDD of its rows, where each row is a local vector. We assume that the number of columns is not huge for a RowMatrix so that a single local vector can be reasonably communicated to the driver and can also be stored / operated on using a single node. An IndexedRowMatrix is similar to a RowMatrix but with row indices, which can be used for identifying rows and executing joins. A CoordinateMatrix is a distributed matrix stored in coordinate list (COO) format, backed by an RDD of its entries. A BlockMatrix is a distributed matrix backed by an RDD of MatrixBlock which is a tuple of (Int, Int, Matrix).
    - RowMatrix
    - IndexedRowMatrix
    - CoordinateMatrix
    A CoordinateMatrix is a distributed matrix backed by an RDD of its entries. Each entry is a tuple of (i: Long, j: Long, value: Double), where i is the row index, j is the column index, and value is the entry value. A CoordinateMatrix should be used only when both dimensions of the matrix are huge and the matrix is very sparse.A CoordinateMatrix can be created from an RDD[MatrixEntry] instance, where MatrixEntry is a wrapper over (Long, Long, Double). A CoordinateMatrix can be converted to an IndexedRowMatrix with sparse rows by calling toIndexedRowMatrix. Other computations for CoordinateMatrix are not currently supported.

        ``` scala
        import org.apache.spark.mllib.linalg.distributed.{CoordinateMatrix, MatrixEntry}
        val entries: RDD[MatrixEntry] = ... // an RDD of matrix entries
        // Create a CoordinateMatrix from an RDD[MatrixEntry].
        val mat: CoordinateMatrix = new CoordinateMatrix(entries)

        // Get its size.
        val m = mat.numRows()
        val n = mat.numCols()

        // Convert it to an IndexRowMatrix whose rows are sparse vectors.
        val indexedRowMatrix = mat.toIndexedRowMatrix()
        ```
    - BlockMatrix
    A BlockMatrix is a distributed matrix backed by an RDD of MatrixBlocks, where a MatrixBlock is a tuple of ((Int, Int), Matrix), where the (Int, Int) is the index of the block, and Matrix is the sub-matrix at the given index with size rowsPerBlock x colsPerBlock. BlockMatrix supports methods such as add and multiply with another BlockMatrix. BlockMatrix also has a helper function validate which can be used to check whether the BlockMatrix is set up properly.
    A BlockMatrix can be most easily created from an IndexedRowMatrix or CoordinateMatrix by calling toBlockMatrix. toBlockMatrix creates blocks of size 1024 x 1024 by default. Users may change the block size by supplying the values through toBlockMatrix(rowsPerBlock, colsPerBlock).
        ``` scala
        import org.apache.spark.mllib.linalg.distributed.{BlockMatrix, CoordinateMatrix, MatrixEntry}

        val entries: RDD[MatrixEntry] = ... // an RDD of (i, j, v) matrix entries
        // Create a CoordinateMatrix from an RDD[MatrixEntry].
        val coordMat: CoordinateMatrix = new CoordinateMatrix(entries)
        // Transform the CoordinateMatrix to a BlockMatrix
        val matA: BlockMatrix = coordMat.toBlockMatrix().cache()

        // Validate whether the BlockMatrix is set up properly. Throws an Exception when it is not valid.
        // Nothing happens if it is valid.
        matA.validate()

        // Calculate A^T A.
        val ata = matA.transpose.multiply(matA)
        ```

### spark矩阵计算实例
1. 一个行向量乘以一个列向量称作向量的内积, 有叫做点积， 结果是一个数
2. 一个列向量乘以一个行向量称作向量的外积，外积是一种特殊的克罗内克积, 结果是一个矩阵
3. 克罗内克乘积 两个任意大小的矩阵, 也被称作直积或者张量积
4. 一般我们说向量不加以指定是指的列向量(也是符合线性方程组) 所以向量就是列向量
5. 两个矩阵相乘可以有4种方式理解 最后归为两类 一种是外积形式或内积形式 $\begin{pmatrix} a_1^T \\\\ a_2^T\end{pmatrix}\begin{pmatrix}b_1 & b_2\end{pmatrix}
$ 这是外积形式
$\begin{pmatrix}a_1 & a_2\end{pmatrix}\begin{pmatrix}b_1^T\\\\b_2^T\end{pmatrix}$ 这是内积形式
内积还是外积主要看形式 左边的向量(括号形式)是列向量还是行向量, 外积是列的形式,内积是行的形式
另外一类是变换
$\begin{pmatrix}a_1&a_2\end{pmatrix}\begin{pmatrix}b_1 & b_2\end{pmatrix}$ 相当于对A进行列变换
$\begin{pmatrix}a_1^T \\\\ a_2^T\end{pmatrix}\begin{pmatrix}b_1^T\\\\ b_2^T\end{pmatrix}$ 相当于对B进行行变换
**所以如果想对一个矩阵进行行变换，可以左乘一个矩阵；相应的如果想对矩阵进行列变换，可以右乘一个矩阵。这种思想被应用到高斯消元的过程中**

7. 分布式方法实现矩阵的乘法可以使用内积法或者外积法, 两种方法的优缺点如下
    - 内积法: shuffle的数据过多。在分布式计算系统中，shuffle过程是影响系统性能的重要因素。但是这种算法在A或B其中一个矩阵是小矩阵的时候是有明显的优势的。例如：如果B是小矩阵，矩阵B可以broadcast到分布式矩阵A的每一个节点上，这么在计算C矩阵的时候，A不需要再进行shuffle操作，能够充分的利用数据本地性进行计算。使用内积法计算分块矩阵的时候 shuffle数量会减少 否则shuffle会多
    - 外积法: 矩阵乘法的外积公式是先计算m个n×k的矩阵，再将m个矩阵相加得到矩阵C。在这个过程中m个矩阵的计算过程是相互独立的。所以能够支持的最大并发量是m。在计算m个矩阵的任意一个的时候，都需要用到A中某一列的n个元素和B中某一行的k个元素，所以在计算m个矩阵的时候，最多需要移动m×(n+k)个元素。在计算完m个矩阵之后，需要将m个矩阵加和。矩阵加法是对应位置元素相加，所以在最后计算C中每一个元素的时候，需要将m个矩阵对应位置的数据shuffle到某一个计算节点进行加和，所以在加和过程中，需要shuffle的最大数据量是m×n×k(因为m个矩阵，每个矩阵有n×k个元素)。每个计算节点最少要计算m个矩阵中的1个，所以计算节点最少需要能够存储n+k个元素的内存，由于每个节点输出元素量为n×k，这个数据量在大规模矩阵中是相当大的，单个计算节点内存很难存储下来，所以每个节点计算的输出结果通常是达到一定量后存储到磁盘上，但是如果是在大规模稀疏矩阵中，m个矩阵中每个矩阵中有值的个数通常都会远小于n×k个，所以第二次shuffle的数据量通常远小于n×k。


8. 关于矩阵的一些重要定义
    - 对称矩阵(Symmetric Matrix) 元素以主对角线为轴对称对应相等的矩阵 $A^T=A$
    - 对角矩阵(Diagonal Matrix) 除对角线外其它元素都为0
    - 三角矩阵(Triangular Matrix) 分为上三角矩阵(主对角线以下元素全为0)和下三角矩阵(主对角线以上元素全为0)
    - 对称矩阵对角化 $\Alpha$是实对称矩阵, $\Lambda$是对角矩阵 $P$是正交矩阵 则
        $P^{-1} \Alpha P=\Lambda$
        对称矩阵的对角化也叫做特征分解(Eigen composition) 谱分解(Spectral Decomposition)
        实对称矩阵的不同特征值对应的特征向量一定正交, 实对称矩阵的特征值一定是实数, 没有复数


#### 实对称矩阵的分解
实对称矩阵具有很重要的意义, 求其特征值和特征向量可以使用Jacobi eigenvalue algorithm 先理解一下概念
- Givens rotation 吉文斯旋转 $G(i, j , \theta)$ $c=\cos\theta$ $s=\sin\theta$  




#### 矩阵分块
对于一个推荐系统的矩阵描述, 用户用大写字母表示, 商品用小写字母表, 假设交互矩阵为x( 0-1矩阵) 那么$x*x^T$是一个itemtoitem的相似矩阵, i行j列的含义是同时交互过商品i和商品j的用户有多少(可以理解成uv
) 可以另主对角线全部为0 
$l*m$ 和矩阵 $m*n$ 相乘 共同的维度m需要被等分, 这样分块后的矩阵才可以再相乘, 所以分块之前可以进行合法性检查
根据[博客](http://hejunhao.me/archives/1503)了解到spark block matrix 在进行矩阵相乘时会有严重性能问题 支持的稀疏矩阵维度就在10000*10000, 所以我会采用博客上介绍的其它方式

1. 基于DataFrame的矩阵乘法
- 基于 DataFrame 方式并非将每一列号作为一个单独字段，而是将矩阵里面的每一个元素看成一个三维数据，即（行号，列号，值），将一个矩阵转化为一个包含三个字段的 table，对于一个稀疏矩阵，只需要记录非零元素，即 100 个非零元素就对应 DataFrame 的 100 行。
- 计算逻辑与 RDD 方式非常相似，主要有三个流程 join => groupby => agg。
- 基于 DataFrame 方式能大幅度提升运算速度，因为 DataFrame 相比 RDD 有更佳的优化支持。


对称矩阵的转置还是对称矩阵 $A^T=A$ 使用余弦相似度可以先norm rescaling 这样最后的稀疏矩阵的每一个元素就是相似度了






2. 归一化与中心化
- 归一化 对数据的数值范围进行特定缩放，但不改变其数据分布的一种线性特征变换
- 归一化的白话表述 把一个数变为一个特定范围 比如(0,1)之间的数 把有量纲表达式变为无量纲
- 归一化的好处有两个:
    * 提升模型的收敛速度
    * 提升模型的精度
    * 深度学习中数据归一化可以防止模型梯度爆炸




在平时的项目中也遇到了不少 case，确实高维稀疏特征的时候，使用 gbdt 很容易过拟合。
但是还是不知道为啥，后来深入思考了一下模型的特点，发现了一些有趣的地方。
假设有1w 个样本， y类别0和1，100维特征，其中10个样本都是类别1，而特征 f1的值为0，1，且刚好这10个样本的 f1特征值都为1，其余9990样本都为0(在高维稀疏的情况下这种情况很常见)，我们都知道这种情况在树模型的时候，很容易优化出含一个使用 f1为分裂节点的树直接将数据划分的很好，但是当测试的时候，却会发现效果很差，因为这个特征只是刚好偶然间跟 y拟合到了这个规律，这也是我们常说的过拟合。但是当时我还是不太懂为什么线性模型就能对这种 case 处理的好？照理说 线性模型在优化之后不也会产生这样一个式子：y = W1*f1 + Wi*fi+….，其中 W1特别大以拟合这十个样本吗，因为反正 f1的值只有0和1，W1过大对其他9990样本不会有任何影响。
后来思考后发现原因是因为现在的模型普遍都会带着正则项，而 lr 等线性模型的正则项是对权重的惩罚，也就是 W1一旦过大，惩罚就会很大，进一步压缩 W1的值，使他不至于过大，而树模型则不一样，树模型的惩罚项通常为叶子节点数和深度等，而我们都知道，对于上面这种 case，树只需要一个节点就可以完美分割9990和10个样本，惩罚项极其之小.
这也就是为什么在高维稀疏特征的时候，线性模型会比非线性模型好的原因了：带正则化的线性模型比较不容易对稀疏特征过拟合。




对于高维稀疏向量最好的相似性计算方法是余弦相似度(相对认可) 并且可以考虑使用 min-max归一化

3. 延生 千万级向量相似度计算
之前我也有遇到这种 **大表关联大表** 问题，如果你的用户特征不是很多，且相似计算的逻辑不是很复杂，那有下面两种做法可供你参考：
第一种是将其中一张表分块，假设你的用户表为A和B，在这里你的A和B其实是一样的，具体为：先对A表切块，可以通过一个map加上一个字段，这个字段是某个范围内的随机数值。通过filter将A表刚刚加的字段过滤出特定的数值。将上面出来的小表与B表做cartesian相似计算操作，剩下的就是一些拼接组合（union）了。
第二种是利用广播的思想：先将A表collect回来，再broadcast出去，这一步如果表很大需要消耗较多的资源和时间。对B表先通过repartition把分区设置大点（建议每个分区处理15-20个用户数据），然后通过map操作和broadcast到每个节点的A表做关联。实际使用过程两种方法我都尝试过，第一种过程中shuffle比较多次，而第二种则一次就ok，但是第二种方法占用的资源比较多，提交时可能需要把executor的内存，num 和 cores都设置的比较充裕。以上上述做法对我这边的性能提升是很高的，但我并不简单的只有这样的操作，由于相似计算过程我实现的算法内部逻辑比较复杂，所以在相似计算前加了一层哈希，其他等等的一些技巧，对最终时间的节约还是很大的，希望能帮到你。


4. 稀疏矩阵的coo csr表达
coo 即矩阵的坐标表达 每一个元素表示为 (row, col, val)
csr 在coo的基础上对于row 又做了一点压缩 由三个一维数组构成 (A, IA, JA)
数组A包含所有的非零元素 从左到右 从上到下
数组IA IA[0] = 0 IA[i] = IA[i-1] + (i-1)th row的非零元素个数 所以从IA可以知道每一行有几个非零元素
数组JA 每一个非零元素的列坐标



## spark 调优
1. 两张表inner join 相同的key 会按照笛卡尔积的形式增加记录条数
2. use the same partitioner 是一种较好的方式 两个要进行join的RDD使用同样的partitioner进行分区减少shuffle
3. two-pass approach
4. co-partitioned - co-located
5. 单个RDD进行aggregateBykey操作 如果有数据倾斜(典型的task执行之间差距太大), 先做repartition, 把数据散列一下
6. repartition 操作 26907192418条 耗时30min
7. shuffle read 和 write 
8. (user size: ,3017289,item size: ,746959) 这种维度的矩阵 677171236754 条记录 资源 40*8*12G + 1 耗时 55min
9. 矩阵(50w * 50W) normalization处理 资源 30g*2core*8g
10. cf iuf方式 15*4core*15G, 15天数据 耗时2h

spark可以较好的支持两个稀疏矩阵的乘法, 但是一旦存在稠密矩阵, 就没有计算优势了, 在ALS算法上已经看到, 矩阵计算缓慢无法应用到实际
关键就在于 拆分出的两个矩阵是稠密矩阵 无法在spark上做矩阵计算，需要找另外的方法
借鉴spark的ALS矩阵分解实现

