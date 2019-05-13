# spell-correction

## 预处理：
--------

处理文本vocab.txt，去除所有标点数字，将单词统一小写得到词汇表VOCAB。

处理文本test_data.txt，分行得到每行的内容，将每行的内容拆分得到每句单词错误数列表wrong_words_num和句子字符串，用nltk.word_tokenize进行字符串分词，得到句子单词列表sentences_origin。

处理文本 channel.txt，统计各种输入错误的次数（编辑距离为1），计算channel
model，存入字典EDIT_P。

通过ngram函数（自己编写）统计unigram和bigram得到一阶二阶语言模型，存入字典UNIGRAM，BIGRAM。（为了探测该拼写检查模型的正确率上限，此处通过ans.txt处理得到n-gram模型而没有用较大的数据集进行n-gram计算，若需要泛化，只需更换ans.txt为一个更大的语料集即可。）

## Word correct:
-------------

**输入：**

待检测句子列表sentences_origin，句子错误数列表wrong_words_num，以及词汇表VOCAB。

**处理：**

将句子列表复制，一份作为原句列表sentences_origin进行对比，一份将单词全部小写作为待处理句子列表sentences。遍历句子列表sentences，得到每个待检验句子sentence。对每条句子sentence进行处理。

*注：Word errors分为两种：none word errors和real word errors。所以spell
correct分为两个主体：none word correct 和 real word correct。*

### 2.1 首先进行None word correct:

**输入：**预处理后的待检测句子sentence

**处理：**遍历句子中的每个单词，判断句子是否为特殊符号，比如：标点、数字等符号，如果是，则直接continue，不对该类型单词检验，如果不是，继续判断单词是否存在词汇表VOCAB中，如果是，则单词不是none
word error，则continue，否则该单词存在拼写错误，进行拼写纠正。分为以为两步：

#### 2.1.1 发掘候选词：

对存在拼写错误的单词进行编辑，通过函数edits1找出所有可能通过一次编辑距离（insert，delete，transform，replace）变换得到的单词，再通过函数known判断单词是否存在于词汇表VOCAB中，将存在的词作为该单词的纠错候选单词集合candidates。对候选集合candidates长度进行判断，若长度为0，说明一次编辑距离范围内无合适候选词，增加至2次编辑距离，通过函数edits2
得到，同样判断是否在VOCAB中，得到最后的结果为该拼写错误的单词的纠错候选词集合candidates。

#### 2.1.2 概率计算：

对候选词集合candidates进行判断找出合适的candidate作为最后的正确纠错单词结果：遍历candidates中的所有candidate，通过函数prob2计算候选词candidate出现的概率p_bigram，对p_bigram进行判断最终得到candidate的概率prob2，通过函数p_type计算channel
model中输入错误的概率(p_type函数只适用于计算edits1编辑距离为1的channel
model)；两者相加prob2+p_type得到最终概率p,选择candidates中概率p最大的candidate作为最终的正确纠错单词。

最后将正确纠错单词的大小写与原句中对应的单词保持一致。并对原句单词进行替换。

### 2.2 继续进行Real word correct：

首先判断该句中是否存在real word error。统计none word
correct中对该句纠正的单词数目wrong，如果wrong小于已知句中错字数wrong_words_num[i]，则视为存在real
word error。

遍历存在real word
error的句子中的每个单词，判断句子是否为特殊符号，比如：标点、数字等符号，如果是，则直接continue，不对该类型单词检验，如果不是，则重复none
word correct
中拼写纠正的发掘候选词步骤，得到候选词集合candidates，唯一不同的是此处candidates包含本身的单词。

对candidates中每个candidate，通过函数prob2计算其在原句中与前一个词以及后一个词的bigram的概率，相乘得到prob2作为该候选词candidate的概率，选择candidates中prob2最大的单词作为最后的正确纠错单词。

最后将正确纠错单词的大小写与原句中对应的单词保持一致。并对原句单词进行替换。

## 结果检验：
----------

将返回的改正后的句子列表写入result.txt中，用eval函数进行结果评估，与ans.txt中的正确结果对比。（edits1表示只考虑编辑距离为\<=1，edits表示只考虑编辑距离\<=2）

| none word correct             | real word correct | 修改大小写 | accuracy |
|-------------------------------|-------------------|------------|----------|
| edits1 + unigram              | \-                | \-         | 40%      |
| edits2 + unigram              | \-                | \-         | 45.7%    |
| edits1 + bigram+channel model | edits2 + bigram   | yes        | 75.7%    |
| edits2 + unigram              | \-                | yes        | 85.8%    |
| edits2 + unigram              | edits2 + bigram   | yes        | 89.2%    |
| edits2 + bigram               | edits2 + bigram   | yes        | 93.2%    |

**总结：**通过输出未正确纠正的句子，可以发现存在以下一些情况：

>   1.由于存在编辑距离大于2的错误单词，因而算法未能检测；

>   2.单词的形态如大小写不一致导致和正确句子不匹配；

>   3.出现了不在字典中的词语因而无法纠正；

>   4.一些特殊符号导致了在某些情况会导致纠错失败等。

**改进：**大小写的转换的完善；特殊符号的处理；句子处理时加上起始符号和终止符号\<start\>和\<end\>；加入更高阶的编辑距离和n-gram模型进行计算；考虑到计算复杂度，暂时只在edits1编辑距离为1时计算了channel
model，可考虑在edits2编辑距离为2时也计算channel model。
