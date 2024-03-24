# Unit 1
## Introduction
What is this course?

- Algorithn: method for solving a problem
- Data structure: method to store information

Topic
<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/png/29674612/1679393721656-1da44e10-bfd8-4632-b787-705cd4e29763.png#averageHue=%23dfdfdf&clientId=udaed0ffc-31e3-4&from=ui&height=262&id=ub4efed70&originHeight=447&originWidth=940&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=40272&status=done&style=none&taskId=ud9e711ae-f393-4ed7-b2c5-498ef7f902d&title=&width=551" width="551" /> </div>


## Union-Find 并查集
### Dynamic Connectivity
在具有$N$个节点的集合，有以下两种操作：

- Union Command：连接连个节点
- Find/connected query：查询两个节点之间是否有通路

动态连接问题的现实应用

- 网络中的计算机
- 人群的社交关系

在进行建模时，可以很方便的将$0$到$N-1$作为数组下标，屏蔽与并查集无关的细节
### Quick-find
用数组存储节点，当下标代表不同节点，当节点对应下标位置存储的元素相同时，两个节点连通
<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/png/29674612/1679489238468-a3f50ebd-5361-48b7-aef3-5b962124b7cc.png#averageHue=%23ededec&clientId=ucf311859-c03d-4&from=ui&height=246&id=u82ce32ce&originHeight=445&originWidth=994&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=25899&status=done&style=none&taskId=u03734d80-a26a-4167-8163-ecc07c203df&title=&width=550" width="550" /> </div>


对于此数据结构，两种操作的具体做法如下：

- Find：检查id是否相同
- Union：假设节点id分别为p、q，遍历id数组使得所有id为q的项的值改为p

具体Java代码如下：
<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/png/29674612/1679489583585-c27541b1-93c6-4bd5-aa0f-6ff52c0b7995.png#averageHue=%23e4e3e3&clientId=ucf311859-c03d-4&from=ui&height=386&id=u9ba9fffa&originHeight=762&originWidth=1090&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=60166&status=done&style=none&taskId=u4fb1f3e3-c531-418e-9669-0adcdc18af6&title=&width=552" width="552" /> </div>



可以看出，union时间复杂度为$O（n）$，对于所有节点进行一次union操作时间复杂度为$O(n^2)$，平凡级算太慢了，对于大型问题是不可接受的
<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/png/29674612/1679489636754-78f5f83c-a1df-4ab0-b330-300b04685194.png#averageHue=%23c4c4c3&clientId=ucf311859-c03d-4&from=ui&height=126&id=ub2a46178&originHeight=182&originWidth=729&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=10793&status=done&style=none&taskId=uab10ba19-3102-4fa9-adb7-635b16c7706&title=&width=503" width="503" /> </div>


> “当计算机变得更快，内存更大时，平方阶算法变得更慢了”
> 计算机大概可以在1s左右访问完一遍主存，这个标准已经保持了大概50-60年（CPU变得更快的时候，主存也在变得越来越大）
> 平凡阶的算法不能成比例的适用于新的技术，当计算机速度和问题规模同时变为原来10倍，平凡阶算法处理时间就要变成原来的10倍

### Quick-union
把数组当成树看待，其中数组每一项中保存其父节点的索引
<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/png/29674612/1679491208422-5b672075-69f6-4fef-80c8-52c17afb6f2c.png#averageHue=%23ecebeb&clientId=ucf311859-c03d-4&from=ui&id=ud9c7df9a&originHeight=265&originWidth=1021&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=17775&status=done&style=none&taskId=u34c22836-eb9b-4a24-81bb-23c6a970c5a&title=" width="600" /> </div>


对于此数据结构，两种操作的具体做法如下：

- Find：往树根遍历，比较根节点是否相同
- Union：假设节点id分别为p、q，将q的根节点作为p的根节点的子节点

Java实现代码：
<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/png/29674612/1679491700744-c9097485-6350-4e3d-ad01-e74747e3f3ee.png#averageHue=%23d8d8d8&clientId=ucf311859-c03d-4&from=ui&height=403&id=ud72738c8&originHeight=765&originWidth=1046&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=65612&status=done&style=none&taskId=ucbb7c735-0339-45f9-9d79-7db7eeb7559&title=&width=551" width="551" /> </div>



快速合并的缺点在于树可能会变得太高，查找的最坏时间复杂度为$O(n)$
### Quick-Union Improvements
改进方法1——加权
通过记录两棵树的大小，避免将大的树作为小的树的根节点的子节点，从而避免让树变高
以下为加权之后的效果：
<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/png/29674612/1679492301215-23d8a529-a9d6-4054-8ae9-8466b66f8539.png#averageHue=%23fefefe&clientId=ucf311859-c03d-4&from=ui&height=282&id=u4127a076&originHeight=579&originWidth=1036&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=46954&status=done&style=none&taskId=u1080ec81-1238-4506-bbaa-03b243417e0&title=&width=505" width="505" /> </div>


Java实现方法，只需做些许改动
<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/png/29674612/1679492398903-8b387cf7-667c-43ab-95cf-5fa12efe98bc.png#averageHue=%23d7d7d7&clientId=ucf311859-c03d-4&from=ui&height=140&id=ud312441f&originHeight=227&originWidth=813&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=16764&status=done&style=none&taskId=u86ea2501-2310-43d8-a8e1-c814046f11d&title=&width=501" width="501" /> </div>



加权之后的算法，对于具有$N$的节点的树，深度最大为$\log _{2}^{N}$
> 

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/png/29674612/1679492804752-4ce922b1-c71f-45e5-a53f-a6f874625943.png#averageHue=%23cfcfcf&clientId=ucf311859-c03d-4&from=ui&height=177&id=u34a3a197&originHeight=218&originWidth=620&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=12560&status=done&style=none&taskId=u691bbf43-35a0-404c-95d3-67edf5cf104&title=&width=502" width="502" /> </div>


