# simhash 库


## simhash 简介

### simhash 计算方法
业界很具有影响力也是使用广泛的大规模相似文章去重的方法是Simhash。早在 2007 年，Simhash 就由 google 工程师 Manku 等在论文 《Detecting Near-Duplicates for Web Crawling》 提了出来，用来解决网络爬虫网页去重的问题。Simhash 实际上一种字符串散列值的计算方法，把一个任意长度的字符串映射为一个64位的整数。其特点是特征相似的字符串计算出的散列值的海明距离较小。对于给定的两篇文章，如果这两篇文章通过simahsh计算出的散列值的海明距离小于等于3，则认为这两篇文章相似。

具体simhash散列值计算步骤为：
1. 首先将文章转换为一组特征权重对集合 {(feature, weight)}，其中 feature 是特征，weight 是其权重。（一般情况下， feature 是文章中的词，weight 是词出现的频次）。
2. 初始化一个 64 维的整数数组 V，数组中每一个元素初始值为 0。
3. 遍历特征集合中的每一个特征权重对，a)利用哈希函数映射 feature 到一个64位的散列值；b)对这个64位的散列值，如果第i比特位上为1，对数组 V 中第 i 个元素加上这个特征的权值weight，否则减去特征的权值 weight。
4. 根据数组 V 中每一个元素的正负符号来确定最终生成的64位散列值，如果数组 V 的第 i 个元素的符号为正，则对应的散列值的第i比特位为1，否则为0。


如果使用有序表来存储 simhash 计算出的散列值，那么增加和删除的时间复杂度都是 O(log(n)) 。而查找相似文章（查找是否存在其他海明距离小于等于3的散列值）的时间复杂度是 O(n)。O(n) 的时间复杂度对于千万量级的文章数来说显然是不可忍受的。不过 simhash 可以通过鸽巢原理来极大降低查询的复杂度。

### simhash 查找方法

**鸽巢原理：** 如果两个simhash散列值的海明距离小于 k ，把64位 simhash 散列值分割成 k+1 组。那么这两个散列值的 k+1 组中，至少存在1组是相等的。
通过鸽巢原理，可以极大程度的降低查找相似文章的时间复杂度。具体方法是 ：

- 在添加 simhash 散列值时，首先对 simhash 散列值进行分组，然后以每一个分组作为键值分别做一个索引存储；
- 在查找相似 simhash 散列值时，首先还是对 simhash 散列值进行分组，然后用每个组分别去对应的索引中查询，最后在把所有组的查询结果合并。这样查找的复杂度可以下降 2^m/(m * (k+1)) 倍，其中m 是每个分组中的比特位数，k 是认为文章相似的最大海明距离。

> 注解：这里这么讲解添加和查找的方法可能不太直观，不少同学都表示没有懂。确实我的描述不是特别清楚。为了让大家搞明白这里，我用大白话举个“栗子”吧！
> - 在添加时，假如我们有一个 64 比特位的 simhash 散列值 H1，我们首先分成4段，每段 16 比特位。也就是把 H1 分段变为 A1 B1 C1 D1。然后我们要有 4 个map（python 里面叫dict），MapA 对应 A1， MapB 对应 B1，MapC 对应 C1，MapD 对应 D1。然后分别以 (A1, H1)、(B1, H1)、(C1, H1)、(D1, H1) 作为键值队把其存入对应的 Map 中。相当对于对每段分别做了独立索引存储。所需要的内存和插入时间都变为 4 倍。
> - 在查找时，同样假如我们有一个 64 比特位的 simhash 散列值 H2，我们还是首先分成4段（分割方式和添加时是一样的），每段 16 比特位。也就是把 H2 分段变为 A2 B2 C2 D2。然后我们要去 4 个 map 中分别进行一次查找，MapA 对应 A2， MapB 对应 B2，MapC 对应 C2，MapD 对应 D2。我们有了 4 份查找结果，最后需要对这 4 份查找结果先取并集然后去重，就得到了最终查找结果。

> 这里使用 map 来举例子只是帮助大家理解，而在实际工程实现的时候并不会真正用 map 这个数据结构。 想了解具体实现的小伙伴可以联系我，我给你看源代码。

当 k 取 3 的时候，$m = 64 / (3 + 1) = 16$，查找的时间复杂度就会下降 $2^{16}/((3+1)*16) = 1024$ 倍，如果进行二级索引，查找的时间复杂度会下降 $2^{16}/((3+1)*16) * 2^{12}/((3+1)*12) = 87381.33$ 倍 ！如何构建索引以及多级索引见下图

![image](https://user-images.githubusercontent.com/80689631/122629500-15b76100-d0f0-11eb-84d0-ced366f52567.png)

好了，到此为止 simhash 的基本原理已经介绍完毕，下面介绍一下这个库怎么使用。

## 快速入门

### 计算字符串simahsh

```cpp

#include "hash.h"
#include "simhash.h"

void TestBuildFromStringFeature()
{
    vector<Simhash::StringFeatureType> features;
    features.push_back(Simhash::StringFeatureType("北京", 1.0));
    features.push_back(Simhash::StringFeatureType("上海", 2.0));
    features.push_back(Simhash::StringFeatureType("成都", 4.3));
    hash_t hash = Simhash::Build(features, JenkinsHash);
}
```

### 查找 simhash 表
```cpp

#include <iostream>
#include <cstdlib>

#include "simhash_table.h"

#include "simhash.h"
#include "hash.h"

#include <cstdlib>
#include <ctime>

int TestSimhashTable()
{
    hash_t h1 = 0x0000000000000000;
    hash_t h2 = 0x0000000000000070;
    hash_t h3 = 0x0000000000000078;

    SimhashTablePtr tablePtr = CreateSimhashTable();
    tablePtr->Insert(h1);
    tablePtr->Insert(h3);

    cout << boolalpha << tablePtr->Search(h1) << endl;

    FindAnswerType ans;
    tablePtr->FindNearDups(h2, ans);

    cout << "ans.size() = " << ans.size() << ". They are :" << endl;
    for (FindAnswerType::iterator iter = ans.begin(); ans.end() != iter; ++iter)
    {
        cout << hex << *iter << endl;
    }
    cout << endl;


    tablePtr->FindNearDups(h1, ans);

    cout << "ans.size() = " << ans.size() << ". They are :" << endl;
    for (FindAnswerType::iterator iter = ans.begin(); ans.end() != iter; ++iter)
    {
        cout << hex << *iter << endl;
    }
    cout << endl;

    //TEST_EQUAL(h1, h3);
    return 0;
}
```

## 参考文献
- [Detecting near-duplicates for web crawling](https://dl.acm.org/doi/abs/10.1145/1242572.1242592)
- [SimHash: Hash-based Similarity Detection](https://www.webrankinfo.com/dossiers/wp-content/uploads/simhash.pdf)
