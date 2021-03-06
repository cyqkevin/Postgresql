本文简单介绍了数据库系统实现中查询优化的关系代数基础,包括优化所基于的关系代数等价规则等.  
查询优化的主要目标是把表达式树变换成等价的表达式树，使得在树中的子表达式生成的关系的平均大小比优化前更小。次要目标是在一个单一查询中，或在要同时求值多于一个查询的时候的所有这些查询中，尽可能形成公共子表达式。

### 一、等价规则

一个关系表达式可以表示成多种形式，前提是这些形式是等价的。如何把一个表达式变换为其他形式的表达式，遵循哪些关系代数规则，下面作一简要描述。

#### 1、选择

**幂等性**  
选择是幂等性的,也就是说多次执行同一个选择运算,跟只执行一次效果一样:  
σ F(R)=σFσFσF(R)

**满足交换律**  
σ F1σF2(R)=σF2σF1(R)

**可分解**  
σ F1∧F2(R)=σF1(σF2(R))=σF2(σF1(R))  
σF1∨F2(R)=σF1(R)∪σF2(R)

**选择下推**  
笛卡尔积耗费的资源巨大，在应用笛卡尔积之前最大可能减少两个关系的大小,把选择下推至参与运算的关系中。  
σ F(R × S)，假设F可以分解为F1、F2、F3，即σF1σF2σF3，F2只与R有关，F3只与S有关，F1与R和S均有关系，那么：  
σF(R × S)=σF1(σF2(R) × σF3(S))

**选择和θ连接**  
σ θ(R × S)= R ⋈θ S  
这其实是θ连接的定义.  
另外,选择运算在下面两个条件下对θ连接运算具有分配律：  
_当选择条件θ 0中的所有属性只涉及参与连接运算的表达式之一时:_  
σθ0(R1 ⋈θ R2) = (σθ0(R1)) ⋈θ R2  
_当选择条件θ 1只涉及R1的属性,θ2只涉及R2的属性时:_  
σθ1∧θ2(R1 ⋈θ R2) =(σθ1(R1)) ⋈θ (σθ2(R2))

**选择和集合运算**  
选择在差、交和并运算上均有分配性:  
σ F(R ∪ S) = σF(R) ∪ σF(S)  
σF(R ∩ S) = σF(R) ∩ σF(S)  
σF(R - S) = σF(R) - σF(S)

**选择和投影**  
选择与投影具有交换性(要求选择中的列是投影字段的子集):  
π a1,a2,...(σF(R))=σF(πa1,a2,...(R))

#### 2、投影

**级联**  
一系列投影运算中只有最后一个运算是必需的,其他的可省略:  
π a1,an,...(πa1,a2,...(πa1,a2,...(R)))=πa1,an,...(R)

**投影和连接**  
令L 1和L2分别代表R1和R2的属性,假设连接条件θ只涉及L1∪L2的属性,则投影在θ连接上具有分配律:  
πL1∪L2(R1 ⋈θ R2) = πL1(R1) ⋈θ πL2(R2)

**投影和集合运算**  
投影在差、交和并运算上均有分配性:  
π a1,a2,...(R ∪ S) = πa1,a2,...(R) ∪ πa1,a2,...(S)  
πa1,a2,...(R ∩ S) = πa1,a2,...(R) ∩ πa1,a2,...(S)  
πa1,a2,...(R - S) = πa1,a2,...(R) - πa1,a2,...(S)

#### 3、连接

**θ(自然)连接满足交换律**  
R ⋈ θ S = S ⋈θ R

**θ连接满足结合律**  
R 1 ⋈θ1 (R2 ⋈θ2∧θ3 R3)=R1 ⋈θ1∧θ3 (R2 ⋈θ2 R3)  
其中θ2只涉及R2和R3的属性.

**自然连接满足结合律**  
R 1 ⋈ (R2 ⋈ R3)=(R1 ⋈ R2) ⋈ R3

#### 4、集合运算

**集合并和交满足交换律**  
R 1 ∪ R2 = R2 ∪ R1  
R1 ∩ R2 = R2 ∩ R1

**集合并和交满足结合律**  
(R 1 ∪ R2) ∪ R3 = R1 ∪ (R2 ∪ R3)  
(R1 ∩ R2) ∩ R2 = R1 ∩ (R2 ∩ R3)

### 二、优化原则

尽可能早地执行选择操作，尽可能在叶子节点完成选择运算；  
尽可能早地执行投影操作，尽可能在叶子节点完成投影运算；  
避免笛卡儿积运算，尽可能把笛卡儿积之前和之后的选择和投影运算合并一起完成。

### 三、案例研究

现有以下三个关系:  
_1、单位信息T_DWXX(以下简称DW)_

DWMC | DWBH | DWDZ  
---|---|---  
X有限公司 | 1001 | 广东省广州市荔湾区  
Y有限公司 | 1002 | 北京市海淀区  
Z有限公司 | 1003 | 广西南宁市五象区  
  
_2、个人信息T_GRXX(以下简称GR)_

DWBH | GRBH | XM | NL  
---|---|---|---  
1001 | 901 | 张三 | 23  
1002 | 902 | 李四 | 33  
1002 | 903 | 王五 | 43  
  
_3、个人缴费信息T_JFXX(以下简称JF)_

GRBH | NY | JE  
---|---|---  
901 | 201801 | 401.30  
901 | 201802 | 401.30  
901 | 201803 | 401.30  
902 | 201801 | 513.10  
902 | 201802 | 513.10  
902 | 201804 | 513.10  
903 | 201801 | 372.22  
903 | 201804 | 372.22  
  
现要求列出单位编号为1001和1002的个人编号、姓名和缴费金额.  
初始结果表达式为(纯粹为了演示需要,把单位信息加入到连接中,实际并不需要)：  
πGRBH,XM,JE(σ(DWBH=1001∨DWBH=1002)∧(DW.DWBH=GR.DWBH)∧(GR.GRBH=JF.GRBH)(DW × GR
× JF))  
转换为语法树:  

![](https://upload-images.jianshu.io/upload_images/8194836-cf74bd8120ae5986.png)

初始表达式

  
DW、GR和JF直接进行笛卡尔积，代价很高，执行选择下推：  
1、选择下推，把查询条件下推：  

![](https://upload-images.jianshu.io/upload_images/8194836-7a1cc455a57c669e.png)

选择下推

  
2、二次选择下推，把选择（连接）下推，右边树形成中间结果  

![](https://upload-images.jianshu.io/upload_images/8194836-9898737a2bcca15b.png)

二次选择下推

  
3、投影下推  

![](https://upload-images.jianshu.io/upload_images/8194836-a7624a8e5d060259.png)

投影下推

  
通过以上转换，减少了连接前的元组数量和参与运算的字段，达到优化目的。

### 四、小结

1、等价规则：关系代数表达式可以遵循等价规则进行转换；  
2、优化：表达式通过等价规则可以改写为更优的等价表达式。

### 五、参考

[维基百科](https://zh.wikipedia.org/wiki/%E5%85%B3%E7%B3%BB%E4%BB%A3%E6%95%B0_\(%E6%95%B0%E6%8D%AE%E5%BA%93\))  
《数据库系统概念》

