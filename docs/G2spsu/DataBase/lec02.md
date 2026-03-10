---
date: 2026-3-10
authors:
    - ZCETHAN
title: lec02 Relational Model
---

关系型数据库是由多种关系组成，具体表现是各种各样的不同的表格，以此体现数据实例之间的关系。

## Structure of Relational Databases

### Basic Structure

关系（relation）的正式定义：对于集合 $D_1,D_2,D_3,\cdots,D_n$，关系 $r$ 被定义为 $D_1\times D_2\times \cdots \times D_n$ 的一个子集。

也就是在所有组合情况中取一种子集的组合。

!!! example "示例"
    ![](.\src\lec02_1.png)

之前说我们用表格的形式来描述关系，那么关系有一些相关的概念来表述表格的一些内容。

- 我们把表格的每一行称为**元组（Tuple）**；
- 把表格的一列称为**属性（Attribute）**，其中每种属性都有其名字（name）；
    - 对于每一种属性，其所有可能的取值称为该属性的**域（domain）**；
    - 对于每一种属性，都要有**原子性（atomic）**，也就是不可分的。

在关系中会出现一些无法填入或者未知的情况，我们会称之为**空（null）**。显然，null 会属于所有的属性的域，而且 null 参与的关系逻辑运算会显得比较复杂，因此我们下面暂时忽略 null 对运算的影响，在最后才引入 null 的影响。

### Concepts about Relation

关系首先关心两个概念：**关系模型（schema）和关系实例（instance）**。二者就像是变量的类型名称和实际值之间的关系。关系模型描述关系的大致结构，而关系实例就是关系中的具体的数据。

关系的性质：
- 元组之间的**顺序**是无关于关系的；
- 同一个关系中不能存在冗余的元组（要**去重**）；
- 属性值拥有**原子性**。

因此一个好的关系型数据库设计，应当能够满足各种范式，从而尽可能减少数据的冗余。

### Key

我们令 $K\subset R$，其中 $R$ 为关系模型。有如下几种情况：

- 如果凭借 $K$ 足以区分所有元组，那么称 $K$ 为 $R$ 的**超码（superkey）**；
- 取所有超码中最小的，称之为**候选码（candidate key）**；
- **主码（primary key）**是一种候选码，是显式的由下划线标出，一般不作修改；
- **外码（foreign key）**是对于两种关系来说的，如果参照关系中的某些属性是被参照关系的主码，那么这些属性就是这一对关系的外码。

## Fundamental Relational-Algebra Operations

### Select

记号：$\sigma_p(r)$

其中，$p$ 是逻辑或算术命题。可以用与或非来连接多个分句。其意义是从关系中选取出满足 $p$ 约束的元组。

??? example "示例"
    ![](.\src\lec02_2.png)

### Porject

记号：$\prod_{A_1,A_2,\dots,A_k}(r)$

其中 $A_i$ 为属性的子集。其意义是仅保留关系中对应属性的列，然后**进行去重**。这一过程与多维向量在广义平面上进行投影的过程十分相似，所以应该不难理解为什么称该操作为投影。

??? example "示例"
    ![](.\src\lec02_3.png)

### Union

记号：$r\cup s$

简单来说就是两个关系的并集。但是要进行并的操作，需要两个关系具有相同的属性数量和兼容的属性内容，简称为**等目相容**。

一般而言，当描述中有 or 等关键字时考虑用 Union。

??? example "示例"
    ![](.\src\lec02_4.png)

### Difference

记号：$r-s$

与集合的差相似，条件运用与并集相似。

??? example "示例"
    ![](.\src\lec02_5.png)

### Cartesian-Product

记号：$r\times s$

笛卡尔积，如果两个关系有相同的属性，则需要对属性分别命名而不能直接合二为一。

??? example "示例"
    ![](.\src\lec02_6.png)

### Rename

记号：$\rho_X(E)$ 或 $\rho_{X(A_1,A_2,\dots,A_n)}(E)$

可以把关系 $E$（以及其所有的属性）更名为 $X$（所有属性 $A_1,A_2,\dots,A_n$）。

### Banking for example

下面我们用银行存储账户的数据库来举例，如下为银行数据库中的关系模型。

![](./src/lec02_7.png)

> E1: Find all loans of over $1200.

使用 Select 进行选取即可：
$$
\sigma_{amount>1200}(loan)
$$

> E2: Find the loan number for each loan of an amount greater than $1200.

由于只需要 loan number，所以作投影即可：

$$
\prod\nolimits_{loan-number}(\sigma_{amount>1200}(loan))
$$

> E3: Find the names of all customers who have a loan, or an account, or both, from the bank.

我们看到了关键字 or，那么显然就是并操作。同时它只需要保留 name，因此：

$$
\prod\nolimits_{customer-name}(borrower)\cup \prod\nolimits_{customer-name}(depositor)
$$

> E4: Find the names of all customers who at least have a loan and an account at bank.

or 变成 and 了，所以直接用交。交前面没讲，是因为交操作可以用并和差来实现。

$$
\prod\nolimits_{customer-name}(borrower)\cap \prod\nolimits_{customer-name}(depositor)
$$

> E5: Find the names of all customers who have a loan at the Perryridge branch.

由于这二者的信息存储在不同的表里，所以想到用笛卡尔积来合并两表的信息。合并之后我们用外码筛选出合法的信息，然后再筛选银行的名字，最后投影到账户名上。

$$
\prod\nolimits_{customer-name}(\sigma_{branch-name='Perryridge'}(\sigma_{borrower.loan-number=loan.loan-number}(borrower\times loan)))
$$

当然我们可以先筛选再做笛卡尔积。此时作积时的表格变得更小了，在这种情况下，复杂度更低。因此我们更倾向于这种写法：

$$
\prod\nolimits_{customer-name}(\sigma_{borrower.loan-number=loan.loan-number}(borrower\times (\sigma_{branch-name='Perryridge'}(loan))))
$$

> E6: Find the largest account balance.

找最大值，我们会想到这需要跨元组进行比较，因此仍然笛卡尔积起手。如何找到最大值呢？我们将所有不是最大值的剔除掉，就剩下最大值了。所以我们对自己的余额
作笛卡尔积，然后筛选出所有前者小于后者的元组，这样前者对应的账户一定不是最大的。而且这个并集是充分的，所以去除这些账户剩下的，就是最大值了。

$$
\prod\nolimits_{balance}(account)-\prod\nolimits_{account.balance}(\sigma_{account.balance<d.balance}(account\times \rho_d(account)))
$$

## Additional Relational-Algebra Operations
## Extended Relational-Algebra Operations
## Modification of the Database