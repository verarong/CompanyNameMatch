# 公司名称匹配

算法概述：基于jieba分词、数百万家企业名称分词后词频统计，提取公司名称中的各类型信息，并依据业务需求计算匹配分值

流程图示意：

![](http://www.weikunt.cn:7788/selif/8yuaigya.png)

* [分析](#分析)

    a.公司名称主体必须完全一致；

    b.地区信息可以有部分省略或缺失，但不可以错误；
      
    c.有分公司信息的，必须全额匹配；
      
    d.其余信息可接受微弱差异
    
 
    
* [算法释义](#算法释义)

    a.基于编辑距离的文本相似度
    
    b.基于jieba分词的文本切分及词性分析
    
    c.单字聚合算法
    
    d.基于词频统计的主体、附加信息提取；根据历史数百万公司名称统计出的词频：[下载](http://www.weikunt.cn:7788/8cf9htec.csv)
    
    e.通过主体、附加、地区、分支信息综合计算匹配度



# 分析

   a.公司名称主体必须完全一致；
    

   b.地区信息可以有部分省略或缺失，但不可以错误；
    
      
   c.有分公司信息的，必须全额匹配；
    
      
   d.其余信息可接受微弱差异


# 算法释义

a.基于编辑距离的文本相似度
```
class StringMatcher:
    def _reset_cache(self):
        self._ratio = self._distance = None
        self._opcodes = self._editops = self._matching_blocks = None

    def __init__(self, seq1='', seq2=''):
        self._str1, self._str2 = seq1, seq2
        self._reset_cache()

    def set_seqs(self, seq1, seq2):
        self._str1, self._str2 = seq1, seq2
        self._reset_cache()

    def set_seq1(self, seq1):
        self._str1 = seq1
        self._reset_cache()

    def set_seq2(self, seq2):
        self._str2 = seq2
        self._reset_cache()

    def get_opcodes(self):
        if not self._opcodes:
            if self._editops:
                self._opcodes = opcodes(self._editops, self._str1, self._str2)
            else:
                self._opcodes = opcodes(self._str1, self._str2)
        return self._opcodes

    def get_editops(self):
        if not self._editops:
            if self._opcodes:
                self._editops = editops(self._opcodes, self._str1, self._str2)
            else:
                self._editops = editops(self._str1, self._str2)
        return self._editops

    def get_matching_blocks(self):
        if not self._matching_blocks:
            self._matching_blocks = matching_blocks(self.get_opcodes(),
                                                    self._str1, self._str2)
        return self._matching_blocks

    def ratio(self):
        if not self._ratio:
            self._ratio = ratio(self._str1, self._str2)
        return self._ratio
    
    //启用长度比较时，如果长度不一致，给予惩罚
    def partial_ratio(self, use_length=False, mismatch_length_point=0.2):
        blocks = self.get_matching_blocks()
        scores = []
        len1, len2 = len(self._str1), len(self._str2)
        # len_ratio = 2 * min(len1, len2) / (len1 + len2) if len1 and len2 else 0
        for block in blocks:
            long_start = block[1] - block[0] if (block[1] - block[0]) > 0 else 0
            long_end = long_start + len(self._str1)
            long_substr = self._str2[long_start:long_end]
            m2 = StringMatcher(self._str1, long_substr)
            r = m2.ratio()
            if use_length:
                scores.append(r - mismatch_length_point)
            else:
                scores.append(r)
        return max(scores)
```
b.基于jieba分词的文本切分及词性分析

```
def normalize(name, branch_words, re_chars):
    if name:
        //清洗各类标点符号
        score = re.sub(r"[%s]+" % re_chars, '', name)
        //jieba分词，不适用HMM模式，不认识的词统一切分为单字
        score = pseg.cut(score, HMM=False)
        words = []
        ns = []
        branch = []
        for x, flag in score:
            if x in branch_words:
                //判定是否为分支机构类信息
                branch.append(x)
            elif flag == 'ns':
                //判定是否为地区信息
                ns.append(x)
            else:
                words.append(x)
        words = join_char(words)
        ns = "".join(ns)
        branch = "".join(branch)
        return words, ns, branch
    return [], "", ""
```

c.单字聚合算法
```
def join_char(words_array):
    //结果存储列表
    score = []  
    //临时列表  
    temp = []
    for word in words_array:
        //非单字
        if len(word) > 1:
            //判断临时列表是否有值，有的话将所有单字拼接并储值至结果列表，并将临时列表清空
            if temp:
                score.append("".join(temp))
                temp = []
            //再存储非单字
            score.append(word)
        else:
            //单字直接存入临时列表
            temp.append(word)
    return score
```

d.基于词频统计的主体、附加信息提取

```
def get_main_sub(string_array, weight_sort_amount):
    if string_array and len(string_array) >= weight_sort_amount:
        weights = [weight.get(x, 0) for x in string_array]
        index = list(map(weights.index, heapq.nsmallest(weight_sort_amount, weights)))[0]
        return string_array[index], "".join([x for i, x in enumerate(string_array) if i != index])
    elif string_array:
        weights = [weight.get(x, 0) for x in string_array]
        index = weights.index(min(weights))
        return string_array[index], "".join([x for i, x in enumerate(string_array) if i != index])
    return "", ""

```
e.通过主体、附加、地区、分支信息综合计算匹配度
```
def match_branch(branch, compare_branch, error_branch):
    //分支机构信息必须全量匹配
    if branch == compare_branch:
        return 0
    else:
        return error_branch


def match_area(area, compare_area, null_area, error_area):
    if area == compare_area:
        return 0
    //当地区信息缺失，或为子串时，扣减字段缺失的分值
    elif not area or not compare_area or area in compare_area or compare_area in area:
        return null_area
    else:
        return error_area


def match_info(words_array, compare_array, mismatch_main, miss_field, weight_sort_amount):
    main, other = get_main_sub(words_array, weight_sort_amount)
    main_, other_ = get_main_sub(compare_array, weight_sort_amount)
    ratio = 1 if main == main_ else mismatch_main
    //当附加信息缺失，扣减字段缺失的分值
    if not other or not other_:
        return ratio - miss_field
    //使用最长相似子串算法，对于长度不一致的，给予少量扣分惩罚
    elif len(other) <= len(other_):
        return ratio - (1 - StringMatcher(other, other_).partial_ratio(True))
    else:
        return ratio - (1 - StringMatcher(other_, other).partial_ratio(True))


def match(name, compare_array, branch_words=('分公司', '分支公司', '支公司', '分店', '分会', '分院', '分部', '分校'),
          re_chars='~`!#$%^&*()_+-/|\';":/.,?><br~·！@#￥%……&*（）——:-=“：’；、。，？\n 》《{}', weight_sort_amount=2,
          mismatch_main=0.3, miss_field=0.1, error_area=0.5, error_branch=0.7):
    score = {}
    //提取主体、地区、分支信息
    name_, area, branch = normalize(name, branch_words, re_chars)
    for i, compare in enumerate(compare_array):
        if name == compare:
            score[i] = 1
        else:
            //提取主体、地区、分支信息
            compare_name, compare_area, compare_branch = normalize(compare, branch_words, re_chars)
            //主体信息匹配度计算
            ratio = match_info(name_, compare_name, mismatch_main, miss_field, weight_sort_amount)
            //地区信息匹配度计算
            ratio_area = match_area(area, compare_area, miss_field, error_area)
            //分支信息匹配度计算
            ratio_branch = match_branch(branch, compare_branch, error_branch)
            //总匹配度计算
            score[i] = max(0, ratio - ratio_area - ratio_branch)
    return score


def match_subject(df_slice):
    //只取末级科目的最后一级信息
    compare_array = [re.sub(".*_", "", x) if "_" in x else "" for x in df_slice.k_kmqc_y]
    //过滤换行后的其他信息
    name = re.sub("<br/>.*", "", df_slice.k_xfdwmc)
    ratio = match(name, compare_array)
    return {df_slice.k_kmqc_y[k]: v for k, v in ratio.items()}
```
