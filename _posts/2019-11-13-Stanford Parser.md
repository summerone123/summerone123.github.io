---
layout:     post
title:     Stanfordparser
subtitle:   大数据处理
date:       2019-11-9
author:     Yi Xia
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - nlp
---
#Stanfordparser
Stanford Parser顾名思义是由斯坦福大学自然语言小组开发的开源句法分析器，是基于概率统计句法分析的一个Java实现。该句法分析器目前提供了5个中文文法的实现。他的优点在于：

- 既是一个高度优化的概率上下文无关文法和词汇化依存分析器，又是一个词汇化上下文无关文法分析器；

- 以权威的并州树库作为分析器的训练数据，支持多语言。目前已经支持英文，中文，德文，意大利文，阿拉伯文等；

- 提供了多样化的分析输出形式，出句法分析树外，还支持分词和词性标注、短语结构、依存关系等输出；

- 内置了分词，词性标注，基于自定义树库的分析器训练等辅助工作。

- 支持多平台，并封装了多种常用语言的接口，例如：java，python，php，ruby等。

基于Stanford Parser的Python接口。由于该句法分析器底层是由java实现，因此使用时需要确保安装JDK。当前，最新的Stanford Parser版本为3.9.1，对JDK的要求是1.8以上。网上JDK的安装教程有很多，可以搜索查看，需要注意的是要配置JAVA_HOME环境变量。

Stanford Parser的Python封装是在nltk库中实现的，因此我们需要安装nltk库。nltk是一款Python的自然语言处理处理工具，但是主要针对英文，对中文效果不好，我们只是用nltk.parse中的Stanford模块。

## 可视化界面
  解压运行lexparser-gui.bat ，这是一个可执行的可视化界面窗口。
  ![实验1](/img/blog_img/%E5%AE%9E%E9%AA%8C1.png)

  
  网上还有一个在线的 stanford Praser，功能更加强大
  [http://nlp.stanford.edu:8080/parser/index.jsp](http://nlp.stanford.edu:8080/parser/index.jsp)
  ![-w1440](/img/blog_img/15736504220927.jpg)

  
## Stanford Parser的实例分析
### python

```python
#分词
import jieba
#导入nltk中斯坦福的分析器
from nltk.parse import stanford
import os

target_str="中国人民解放军信息工程大学以原信息工程大学、外国语学院为基础重建，隶属战略支援部队，校区位于河南省郑州市、洛阳市，担负着为国防和军队现代化建设培养信息领域高层次人才的重任。学校有80余年的办学历史，前身是军委工程学校第二部、第三部和东北民主联军测绘学校，先后为国家和军队培养20余万高素质人才。学校是国务院学位委员会授权的博士、硕士学位授予单位，是首批国家一流网络安全学院建设示范项目高校，军队唯一的国家网络安全人才培养基地；是国家非通用语人才培养基地、全军出国人员外语培训基地、外国军事留学生汉语培训基地。"
seg_list=jieba.cut(target_str,cut_all=False,HMM=True)
#用空格再重新拼接分词的结果，因为Stanford Parser的句法分析器接受的输入是分词完后以空格隔开的句子。
seg_str=' '.join(seg_list)
root='/Users/summerone/Desktop/NLP/Syntactic\ analysis/stanford-parser-full-2018-10-17 '
#分析器的路径
parser_path="/Users/summerone/Desktop/NLP/Syntactic analysis/stanford-parser-full-2018-10-17/stanford-parser.jar"
model_path="/Users/summerone/Desktop/NLP/Syntactic analysis/stanford-parser-full-2018-10-17/stanford-parser-3.9.2-models.jar"
#指定JDK路径，如果没有配置JAVA_HOME系统设置
if not os.environ.get('JAVA_HOME'):
    #设置jdk地址
    JAVA_HOME="/Library/Java/JavaVirtualMachines/jdk1.8.0_211"
    os.environ['JAVA_HOME']=JAVA_HOME
#PCFG模型路径
pcfg_path='edu/stanford/nlp/models/lexparser/chinesePCFG.ser.gz'
#配置解析器
parser=stanford.StanfordParser(
    path_to_jar=parser_path,
    path_to_models_jar=model_path,
    model_path=pcfg_path
    )
#分析指定数据,返回一个generator
sentence=parser.raw_parse(seg_str)
print(sentence)
for line in sentence:
    #需要注意的是line类型为<class 'nltk.tree.Tree'>
    print(line)
    #画树
    line.draw()

```


```
<list_iterator object at 0x10d331860>
(ROOT
  (IP
    (IP
      (NP
        (NR 中国)
        (NN 人民)
        (NN 解放军信息工程大学)
        (NN 以原)
        (NN 信息工程)
        (NN 大学)
        (PU 、)
        (NN 外国语)
        (NN 学院))
      (VP
        (VP (PP (P 为) (NP (NN 基础))) (VP (VV 重建)))
        (PU ，)
        (VP (VV 隶属) (NP (NN 战略) (NN 支援) (NN 部队)))))
    (PU ，)
    (IP
      (NP (NN 校区))
      (VP
        (VP
          (VV 位于)
          (NP (NP (NP (NR 河南省)) (NP (NN 郑州市))) (PU 、) (NP (NN 洛阳市))))
        (PU ，)
        (VP
          (VV 担负)
          (AS 着)
          (NP
            (CP
              (IP
                (VP
                  (PP
                    (P 为)
                    (NP
                      (NP (NN 国防) (CC 和) (NN 军队) (NN 现代化))
                      (NP (NN 建设))))
                  (VP (VV 培养) (NP (NN 信息) (NN 领域) (NN 高层次) (NN 人才)))))
              (DEC 的))
            (NP (NN 重任))))))
    (PU 。)
    (IP
      (NP (NN 学校))
      (VP
        (VE 有)
        (NP
          (DNP (NP (QP (CD 80)) (NP (NN 余年))) (DEG 的))
          (NP (NN 办学) (NN 历史)))))
    (PU ，)
    (IP
      (NP (NN 前身))
      (VP
        (VC 是)
        (NP
          (NP
            (NP (NN 军委) (NN 工程))
            (NP (NN 学校))
            (QP (OD 第二部) (PU 、) (OD 第三部)))
          (CC 和)
          (NP (NN 东北民主联军) (NN 测绘) (NN 学校)))))
    (PU ，)
    (IP
      (VP
        (ADVP (AD 先后))
        (PP (P 为) (NP (NN 国家) (CC 和) (NN 军队)))
        (VP
          (VV 培养)
          (NP
            (QP (CD 20))
            (NP (QP (CD 余万)) (NP (NN 高素质)))
            (NP (NN 人才))))))
    (PU 。)
    (IP
      (NP (NN 学校))
      (VP
        (VC 是)
        (IP
          (NP
            (DNP (NP (NN 国务院学位委员会) (NN 授权)) (DEG 的))
            (NP (NN 博士) (PU 、) (NN 硕士学位)))
          (VP (VV 授予) (NP (NN 单位))))))
    (PU ，)
    (IP
      (VP
        (VC 是)
        (NP
          (NP
            (NP (NN 首批) (NN 国家))
            (ADJP (JJ 一流))
            (NP (NN 网络安全) (NN 学院) (NN 建设) (NN 示范)))
          (NP (NN 项目) (NN 高校)))))
    (PU ，)
    (IP
      (NP (NN 军队))
      (NP (DNP (ADJP (JJ 唯一)) (DEG 的)) (NP (NN 国家) (NN 网络安全)))
      (VP
        (VV 人才培养)
        (NP
          (NP (NN 基地))
          (PU ；)
          (PRN
            (VP
              (VP (VC 是) (NP (NN 国家)))
              (VP
                (VC 非)
                (NP
                  (NR 通用)
                  (NN 语)
                  (NN 人才培养)
                  (NN 基地)
                  (PU 、)
                  (NN 全军)
                  (NN 出国人员)
                  (NN 外语)
                  (NN 培训基地)
                  (PU 、)
                  (NN 外国)
                  (NN 军事)
                  (NN 留学生)
                  (NN 汉语)
                  (NN 培训基地))))))))
    (PU 。)))

```
![-w1547](/img/blog_img/15736227815996.jpg)


### java
java 实现了一下 Stanford Praser

```java
import java.util.*;
import java.io.StringReader;
 
 
import edu.stanford.nlp.process.CoreLabelTokenFactory;
import edu.stanford.nlp.process.DocumentPreprocessor;
import edu.stanford.nlp.process.PTBTokenizer;
import edu.stanford.nlp.process.TokenizerFactory;
import edu.stanford.nlp.ling.CoreLabel;  
import edu.stanford.nlp.ling.HasWord;  
import edu.stanford.nlp.trees.*;
import edu.stanford.nlp.parser.lexparser.LexicalizedParser;
 
class ParserDemo {
 
  public static void main(String[] args) {
    LexicalizedParser lp = 
       LexicalizedParser.loadModel("edu/stanford/nlp/models/lexparser/chinesePCFG.ser.gz");
    if (args.length > 0) {
      demoDP(lp, args[0]);
    } else {
      demoAPI(lp);
    }
  }
 
  public static void demoDP(LexicalizedParser lp, String filename) {
    // This option shows loading and sentence-segment and tokenizing
    // a file using DocumentPreprocessor
    TreebankLanguagePack tlp = new PennTreebankLanguagePack();
    GrammaticalStructureFactory gsf = tlp.grammaticalStructureFactory();
    // You could also create a tokenier here (as below) and pass it
    // to DocumentPreprocessor
    for (List<HasWord> sentence : new DocumentPreprocessor(filename)) {
      Tree parse = lp.apply(sentence);
      parse.pennPrint();
      System.out.println();
      
      GrammaticalStructure gs = gsf.newGrammaticalStructure(parse);
      Collection tdl = gs.typedDependenciesCCprocessed(true);
      System.out.println(tdl);
      System.out.println();
    }
  }
 
  public static void demoAPI(LexicalizedParser lp) {
    // This option shows parsing a list of correctly tokenized words
    String[] sent = { "我", "是", "一名", "好", "学生", "。" };
    List<CoreLabel> rawWords = new ArrayList<CoreLabel>();
    for (String word : sent) {
      CoreLabel l = new CoreLabel();
      l.setWord(word);
      rawWords.add(l);
    }
    Tree parse = lp.apply(rawWords);
    parse.pennPrint();
    System.out.println();
 
    TreebankLanguagePack tlp = lp.getOp().langpack();
    GrammaticalStructureFactory gsf = tlp.grammaticalStructureFactory();
    GrammaticalStructure gs = gsf.newGrammaticalStructure(parse);
    List<TypedDependency> tdl = gs.typedDependenciesCCprocessed();
    System.out.println(tdl);
    System.out.println();
  
    TreePrint tp = new TreePrint("penn,typedDependenciesCollapsed",tlp);
    tp.printTree(parse);
  }
 
  private ParserDemo() {} // static methods only
 
}
```


##附录
计算机语言学家罗宾森总结了依存语法的四条定理：

1、一个句子中存在一个成分称之为根（root），这个成分不依赖于其它成分。

2、其它成分直接依存于某一成分；

3、任何一个成分都不能依存与两个或两个以上的成分；

4、如果A成分直接依存于B成分，而C成分在句中位于A和B之间，那么C或者直接依存于B，或者直接依存于A和B之间的某一成分；

5、中心成分左右两面的其它成分相互不发生关系。

使用斯坦福句法分析器做依存句法分析可以输出句子的依存关系，Stanford parser基本上是一个词汇化的概率上下文无关语法分析器，同时也使用了依存分析。

下面是对分析的结果中一些符号的解释：

ROOT：要处理文本的语句；IP：简单从句；NP：名词短语；VP：动词短语；PU：断句符，通常是句号、问号、感叹号等标点符号；LCP：方位词短语；PP：介词短语；CP：由‘的’构成的表示修饰性关系的短语；DNP：由‘的’构成的表示所属关系的短语；ADVP：副词短语；ADJP：形容词短语；DP：限定词短语；QP：量词短语；NN：常用名词；NR：固有名词；NT：

ROOT：要处理文本的语句

IP：简单从句

NP：名词短语

VP：动词短语

PU：断句符，通常是句号、问号、感叹号等标点符号

LCP：方位词短语

PP：介词短语

CP：由‘的’构成的表示修饰性关系的短语

DNP：由‘的’构成的表示所属关系的短语

ADVP：副词短语

ADJP：形容词短语

DP：限定词短语

QP：量词短语

NN：常用名词

NR：固有名词

NT：时间名词

PN：代词

VV：动词

VC：是

CC：表示连词

VE：有

VA：表语形容词

AS：内容标记（如：了）

VRD：动补复合词

CD: 表示基数词

DT: determiner 表示限定词

EX: existential there 存在句

FW: foreign word 外来词

IN: preposition or conjunction, subordinating 介词或从属连词

JJ: adjective or numeral, ordinal 形容词或序数词

JJR: adjective, comparative 形容词比较级

JJS: adjective, superlative 形容词最高级

LS: list item marker 列表标识

MD: modal auxiliary 情态助动词

PDT: pre-determiner 前位限定词

POS: genitive marker 所有格标记

PRP: pronoun, personal 人称代词

RB: adverb 副词

RBR: adverb, comparative 副词比较级

RBS: adverb, superlative 副词最高级

RP: particle 小品词

SYM: symbol 符号

TO:”to” as preposition or infinitive marker 作为介词或不定式标记

WDT: WH-determiner WH限定词

WP: WH-pronoun WH代词

WP$: WH-pronoun, possessive WH所有格代词

WRB:Wh-adverb WH副词

关系表示

abbrev: abbreviation modifier，缩写

acomp: adjectival complement，形容词的补充；

advcl : adverbial clause modifier，状语从句修饰词

advmod: adverbial modifier状语

agent: agent，代理，一般有by的时候会出现这个

amod: adjectival modifier形容词

appos: appositional modifier,同位词

attr: attributive，属性

aux: auxiliary，非主要动词和助词，如BE,HAVE SHOULD/COULD等到

auxpass: passive auxiliary 被动词

cc: coordination，并列关系，一般取第一个词

ccomp: clausal complement从句补充

complm: complementizer，引导从句的词好重聚中的主要动词

conj : conjunct，连接两个并列的词。

cop: copula。系动词（如be,seem,appear等），（命题主词与谓词间的）连系

csubj : clausal subject，从主关系

csubjpass: clausal passive subject 主从被动关系

dep: dependent依赖关系

det: determiner决定词，如冠词等

dobj : direct object直接宾语

expl: expletive，主要是抓取there

infmod: infinitival modifier，动词不定式

iobj : indirect object，非直接宾语，也就是所以的间接宾语；

mark: marker，主要出现在有“that” or “whether”“because”, “when”,

mwe: multi-word expression，多个词的表示

neg: negation modifier否定词

nn: noun compound modifier名词组合形式

npadvmod: noun phrase as adverbial modifier名词作状语

nsubj : nominal subject，名词主语

nsubjpass: passive nominal subject，被动的名词主语

num: numeric modifier，数值修饰

number: element of compound number，组合数字

parataxis: parataxis: parataxis，并列关系

partmod: participial modifier动词形式的修饰

pcomp: prepositional complement，介词补充

pobj : object of a preposition，介词的宾语

poss: possession modifier，所有形式，所有格，所属

possessive: possessive modifier，这个表示所有者和那个’S的关系

preconj : preconjunct，常常是出现在 “either”, “both”, “neither”的情况下

predet: predeterminer，前缀决定，常常是表示所有

prep: prepositional modifier

prepc: prepositional clausal modifier

prt: phrasal verb particle，动词短语

punct: punctuation，这个很少见，但是保留下来了，结果当中不会出现这个

purpcl : purpose clause modifier，目的从句

quantmod: quantifier phrase modifier，数量短语

rcmod: relative clause modifier相关关系

ref : referent，指示物，指代

rel : relative

root: root，最重要的词，从它开始，根节点

tmod: temporal modifier

xcomp: open clausal complement

xsubj : controlling subject 掌控者

中心语为谓词

subj — 主语

nsubj — 名词性主语（nominal subject） （同步，建设）

top — 主题（topic） （是，建筑）

npsubj — 被动型主语（nominal passive subject），专指由“被”引导的被动句中的主语，一般是谓词语义上的受事 （称作，镍）

csubj — 从句主语（clausal subject），中文不存在

xsubj — x主语，一般是一个主语下面含多个从句 （完善，有些）

中心语为谓词或介词

obj — 宾语

dobj — 直接宾语 （颁布，文件）

iobj — 间接宾语（indirect object），基本不存在

range — 间接宾语为数量词，又称为与格 （成交，元）

pobj — 介词宾语 （根据，要求）

lobj — 时间介词 （来，近年）

中心语为谓词

comp — 补语

ccomp — 从句补语，一般由两个动词构成，中心语引导后一个动词所在的从句(IP) （出现，纳入）

xcomp — x从句补语（xclausal complement），不存在

acomp — 形容词补语（adjectival complement）

tcomp — 时间补语（temporal complement） （遇到，以前）

lccomp — 位置补语（localizer complement） （占，以上）

— 结果补语（resultative complement）

中心语为名词

mod — 修饰语（modifier）

pass — 被动修饰（passive）

tmod — 时间修饰（temporal modifier）

rcmod — 关系从句修饰（relative clause modifier） （问题，遇到）

numod — 数量修饰（numeric modifier） （规定，若干）

ornmod — 序数修饰（numeric modifier）

clf — 类别修饰（classifier modifier） （文件，件）

nmod — 复合名词修饰（noun compound modifier） （浦东，上海） amod — 形容词修饰（adjetive modifier） （情况，新）

advmod — 副词修饰（adverbial modifier） （做到，基本）

vmod — 动词修饰（verb modifier，participle modifier）

prnmod — 插入词修饰（parenthetical modifier）

neg — 不定修饰（negative modifier） (遇到，不)

det — 限定词修饰（determiner modifier） （活动，这些） possm — 所属标记（possessive marker），NP

poss — 所属修饰（possessive modifier），NP

dvpm — DVP标记（dvp marker），DVP （简单，的）

dvpmod — DVP修饰（dvp modifier），DVP （采取，简单）

assm — 关联标记（associative marker），DNP （开发，的）

assmod — 关联修饰（associative modifier），NP|QP （教训，特区） prep — 介词修饰（prepositional modifier） NP|VP|IP（采取，对） clmod — 从句修饰（clause modifier） （因为，开始）

plmod — 介词性地点修饰（prepositional localizer modifier） （在，上） asp — 时态标词（aspect marker） （做到，了）

partmod– 分词修饰（participial modifier） 不存在

etc — 等关系（etc） （办法，等）

中心语为实词

conj — 联合(conjunct)

cop — 系动(copula) 双指助动词？？？？

cc — 连接(coordination)，指中心词与连词 （开发，与）

其它

attr — 属性关系 （是，工程）

cordmod– 并列联合动词（coordinated verb compound） （颁布，实行） mmod — 情态动词（modal verb） （得到，能）

ba — 把字关系

tclaus — 时间从句 （以后，积累）

— semantic dependent

cpm — 补语化成分（complementizer），一般指“的”引导的CP （振兴，的）
