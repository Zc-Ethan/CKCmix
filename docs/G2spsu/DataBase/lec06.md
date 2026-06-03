---
date: 2026-4-12
authors:
    - ZCETHAN
title: lec06 Relational Database Design
---

在上一章中，我们从 ER 图出发，将现实世界的实体和联系转化为了关系模式。现在的问题是：这样的关系模式是不是足够好？如果一个关系模式设计得不好，就会出现数据冗余、插入异常、删除异常和更新异常。本章讨论的就是如何用函数依赖、多值依赖和范式理论来判断并改进关系数据库设计。

## First Normal Form

我们先从最基本的第一范式（First Normal Form, 1NF）开始。一个域是**原子的**（atomic），意思是数据库不再把其中的值继续拆分使用。若一个关系模式 $R$ 的所有属性域都是原子的，那么 $R$ 就满足第一范式。

!!! note "1NF 的直观理解"
    关系数据库默认要求所有关系都在 1NF 中。也就是说，表格中的一个单元格应该是一个普通值，而不是一个集合、列表、嵌套对象或复杂结构。

下面这些都属于非原子值的典型情况：

- 复合属性：比如把完整姓名作为一个字段，但之后又需要拆分出姓和名；
- 多值属性：比如一个人的所有电话号码都放在同一个字段中；
- 复杂数据类型：比如面向对象数据库中的对象、集合等结构。

如果遇到非原子属性，我们一般有几种处理方式：

=== "复合属性"
    把复合属性拆成多个属性。比如：

    ```text
    customer(customer-id, first-name, last-name, street, city)
    ```

=== "多值属性"
    更推荐的方法是单独建表：

    ```text
    person(pname, ...)
    phone(pname, phone-number)
    ```

=== "不推荐做法"
    把多个电话写成 `phone1, phone2, phone3`，或者把所有电话塞进一个 `phones` 字段。这样会让查询、更新和约束检查都变复杂。

需要注意的是，原子性并不是单纯由数据类型决定的，而是由**使用方式**决定的。比如字符串通常可以看成原子值，但如果学号形如 `CS0012`，并且应用程序会取前两个字符来判断院系，那么这个字符串就不再是真正的原子值。更好的做法是把院系作为独立属性存进数据库，而不是把信息编码进字符串中。

## Pitfalls in Relational Database Design

关系数据库设计的目标是找到一组“好”的关系模式。坏的设计通常会导致：

- 冗余存储：同一事实被重复记录多次；
- 更新异常：修改一个事实时需要改很多行，容易不一致；
- 插入异常：想插入某些事实时，因为缺少其它事实而无法插入；
- 删除异常：删除某些事实时，连带丢失了本来还想保留的信息。

考虑如下关系模式：

```text
Lending-schema = (
    branch-name, branch-city, assets,
    customer-name, loan-number, amount
)
```

假设它的主键是 `(branch-name, customer-name, loan-number)`。这张表把支行信息、客户信息和贷款信息混在了一起。对于同一个支行，它的 `branch-city` 和 `assets` 会在每一笔贷款中重复出现。

!!! warning "这个模式的问题"
    如果某个支行的资产发生变化，那么所有涉及该支行贷款的元组都要修改。只要漏改一行，就会产生不一致。更麻烦的是，如果一个支行暂时没有任何贷款，我们可能无法自然地保存这个支行的信息，除非引入大量 null 值。

一种自然的改进方式是**分解**（decomposition）：

```text
Branch-schema = (branch-name, branch-city, assets)
Loan-info-schema = (branch-name, customer-name, loan-number, amount)
```

分解的基本要求有两个。首先，原模式中的所有属性都必须出现在分解后的模式中。其次，更重要的是，分解应该是**无损连接分解**（lossless-join decomposition），也就是分解之后再自然连接回来，不能多出或少掉元组。

!!! example "一个有损分解"
    如果 $R(A,B)$ 被分解成 $R_1(A)$ 和 $R_2(B)$，那么投影后再连接得到的是 $A$ 和 $B$ 的笛卡尔组合。这个结果通常会比原来的 $R$ 多出许多元组，因此不是无损连接分解。

## Functional Dependencies

为了形式化地判断一个关系模式是否“好”，我们需要引入**函数依赖**（Functional Dependency, FD）。

设 $R$ 是一个关系模式，$\alpha\subseteq R,\beta\subseteq R$。如果对于 $R$ 上的任何合法关系 $r$，任意两个元组 $t_1,t_2$ 只要满足：

$$
t_1[\alpha]=t_2[\alpha]
$$

就一定满足：

$$
t_1[\beta]=t_2[\beta]
$$

那么我们说函数依赖 $\alpha\to\beta$ 在 $R$ 上成立。也可以说，$\alpha$ 函数决定 $\beta$，或者 $\beta$ 函数依赖于 $\alpha$。

!!! note "FD 和具体实例的区别"
    某个具体关系实例满足一个函数依赖，并不代表该函数依赖就在关系模式上成立。模式上的函数依赖来自语义约束，而不是单纯从某一张表当前的数据中“猜”出来的。

函数依赖可以看成键的推广：

- $K$ 是 $R$ 的超键，当且仅当 $K\to R$；
- $K$ 是 $R$ 的候选键，当且仅当 $K\to R$，并且不存在真子集 $\alpha\subset K$ 满足 $\alpha\to R$；
- 函数依赖还可以表达许多键无法表达的约束。

比如：

```text
Loan-info-schema = (customer-name, loan-number, branch-name, amount)
F = {
    loan-number -> amount,
    loan-number -> branch-name,
    (customer-name, loan-number) -> amount,
    (customer-name, loan-number) -> branch-name
}
```

这里我们通常不希望 `loan-number -> customer-name` 成立，因为一笔贷款可能有多个借款人。

### Trivial Dependency

如果一个函数依赖对所有关系都成立，那么它就是**平凡函数依赖**。一般地：

$$
\alpha\to\beta\text{ is trivial}\iff \beta\subseteq\alpha
$$

例如：

```text
A -> A
AB -> A
(customer-name, loan-number) -> customer-name
```

## Closure of Functional Dependencies

给定函数依赖集 $F$，我们可以从 $F$ 推出更多函数依赖。所有能由 $F$ 逻辑推出的函数依赖集合，称为 $F$ 的闭包，记作 $F^+$。

例如：

```text
F = {A -> B, B -> C}
```

那么 $A\to C$ 也应该在 $F^+$ 中。

### Armstrong's Axioms

Armstrong 公理提供了一组求闭包的推理规则：

| 规则 | 形式 | 说明 |
| --- | --- | --- |
| 自反律 | 若 $\beta\subseteq\alpha$，则 $\alpha\to\beta$ | 得到平凡依赖 |
| 增补律 | 若 $\alpha\to\beta$，则 $\gamma\alpha\to\gamma\beta$ | 两边同时加属性 |
| 传递律 | 若 $\alpha\to\beta,\beta\to\gamma$，则 $\alpha\to\gamma$ | 依赖传递 |

这组公理是 sound 且 complete 的。也就是说，它推出的依赖都是真的，并且所有真的依赖最终都可以推出。

在手算时，我们还经常使用一些由 Armstrong 公理推出的补充规则：

- 合并律：若 $\alpha\to\beta$ 且 $\alpha\to\gamma$，则 $\alpha\to\beta\gamma$；
- 分解律：若 $\alpha\to\beta\gamma$，则 $\alpha\to\beta$ 且 $\alpha\to\gamma$；
- 伪传递律：若 $\alpha\to\beta$ 且 $\gamma\beta\to\delta$，则 $\alpha\gamma\to\delta$。

!!! example "闭包推导"
    设：

    ```text
    R = (A, B, C, G, H, I)
    F = {A -> B, A -> C, CG -> H, CG -> I, B -> H}
    ```

    可以推出：

    - $A\to H$，因为 $A\to B$ 且 $B\to H$；
    - $AG\to I$，因为 $A\to C$，所以 $AG\to CG$，又有 $CG\to I$；
    - $CG\to HI$，因为 $CG\to H$ 且 $CG\to I$。

## Closure of Attribute Sets

直接求 $F^+$ 代价很大，因为可能的函数依赖数量随属性数指数增长。实际中我们更常用**属性集闭包**。

给定属性集 $\alpha$ 和函数依赖集 $F$，$\alpha$ 在 $F$ 下的闭包记作 $\alpha^+$，表示所有能由 $\alpha$ 函数决定的属性集合。

!!! tip "属性闭包的两个最常用判断"
    - 判断 $\alpha$ 是否是超键：看 $R\subseteq\alpha^+$ 是否成立；
    - 判断 $\alpha\to\beta$ 是否成立：看 $\beta\subseteq\alpha^+$ 是否成立。

属性闭包的算法如下：

```text
result := alpha
while result changes:
    for each beta -> gamma in F:
        if beta is contained in result:
            result := result union gamma
alpha+ := result
```

!!! example "计算属性闭包"
    设：

    ```text
    R = (A, B, C, G, H, I)
    F = {A -> B, A -> C, CG -> H, CG -> I, B -> H}
    ```

    计算 $(AG)^+$：

    1. 初始 `result = AG`；
    2. 由 `A -> B` 和 `A -> C`，得到 `ABCG`；
    3. 由 `CG -> H`，得到 `ABCGH`；
    4. 由 `CG -> I`，得到 `ABCGHI`。

    因此 $(AG)^+=R$，所以 `AG` 是超键。又因为 $A^+=ABCH$，$G^+=G$，二者都不是超键，所以 `AG` 是候选键。

## Canonical Cover

数据库在更新时需要检查函数依赖是否被破坏。如果函数依赖集 $F$ 太大，检查成本会很高。因此我们希望找到一个与 $F$ 等价但更小的依赖集，称为**正则覆盖**（Canonical Cover），记作 $F_c$。

直观地说，$F_c$ 是一个最小化的函数依赖集，满足：

- $F_c$ 与 $F$ 等价，即二者能相互推出；
- 没有冗余的函数依赖；
- 函数依赖左右两边没有无关属性；
- 左部相同的依赖会尽量合并。

### Extraneous Attributes

如果一个属性从函数依赖中删掉后，不改变整个依赖集表达的约束，那么它就是**无关属性**（extraneous attribute）。

=== "左部无关属性"
    对于 $\alpha\to\beta$，如果 $A\in\alpha$，并且用 $(\alpha-A)\to\beta$ 替换原依赖后，仍然与原依赖集等价，那么 $A$ 在左部是无关属性。

    测试方法：计算 $(\alpha-A)^+$，如果包含 $\beta$，则 $A$ 可删。

=== "右部无关属性"
    对于 $\alpha\to\beta$，如果 $A\in\beta$，令：

    $$
    F'=(F-\{\alpha\to\beta\})\cup\{\alpha\to(\beta-A)\}
    $$

    如果在 $F'$ 下仍能推出 $\alpha\to A$，那么 $A$ 在右部是无关属性。

!!! example "正则覆盖计算"
    设：

    ```text
    R = (A, B, C)
    F = {A -> BC, B -> C, A -> B, AB -> C}
    ```

    先合并左部相同的依赖，得到：

    ```text
    F' = {A -> BC, B -> C, AB -> C}
    ```

    `AB -> C` 中的 `A` 是无关的，因为已有 `B -> C`。删去后又发现 `A -> BC` 中的 `C` 是无关的，因为 `A -> B` 且 `B -> C`。所以最终：

    ```text
    Fc = {A -> B, B -> C}
    ```

## Decomposition

规范化的核心技术就是分解。对于一个关系模式 $R$，我们希望把它分解为若干关系模式 $R_1,R_2,\dots,R_n$，并尽量满足：

- 无损连接分解；
- 依赖保持；
- 每个子模式都处于足够好的范式，如 BCNF 或 3NF。

### Lossless Join

对于二元分解 $R\to R_1,R_2$，无损连接的判别条件非常常用：

$$
R_1\cap R_2\to R_1
$$

或：

$$
R_1\cap R_2\to R_2
$$

只要其中一个依赖在 $F^+$ 中成立，该分解就是无损连接分解。

!!! note
    通俗地说，两个子模式的公共属性必须能作为其中至少一边的键，这样连接时才不会产生额外组合。

### Dependency Preservation

依赖保持的意义是：我们可以在各个子关系上分别检查函数依赖，而不需要先把所有子关系连接回原关系。

设 $F_i$ 是 $F^+$ 中只涉及 $R_i$ 属性的函数依赖集合。如果：

$$
(F_1\cup F_2\cup\cdots\cup F_n)^+=F^+
$$

那么该分解是依赖保持的。

!!! example "无损但不一定保持依赖"
    设：

    ```text
    R = (A, B, C)
    F = {A -> B, B -> C}
    ```

    分解为 `R1(A,B), R2(B,C)` 时，公共属性为 `B`，且 `B -> C`，所以无损；同时 `A -> B` 和 `B -> C` 分别保留在两个子模式中，所以依赖保持。

    但分解为 `R1(A,B), R2(A,C)` 时，公共属性为 `A`，且 `A -> B`，所以仍然无损；可是 `B -> C` 无法在任何单个子模式中检查，因此不保持依赖。

## Boyce-Codd Normal Form

BCNF（Boyce-Codd Normal Form）是一个比较强的范式。关系模式 $R$ 在函数依赖集 $F$ 下满足 BCNF，当且仅当对于 $F^+$ 中每一个形如 $\alpha\to\beta$ 的函数依赖，至少满足以下条件之一：

1. $\alpha\to\beta$ 是平凡依赖；
2. $\alpha$ 是 $R$ 的超键。

换句话说，在 BCNF 中，任何非平凡函数依赖的左部都必须是超键。

!!! example
    设：

    ```text
    R = (A, B, C)
    F = {A -> B, B -> C}
    ```

    由于 $A^+=ABC$，所以 `A` 是键。但是对于 `B -> C`，`B` 不是键，因此 $R$ 不满足 BCNF。可以分解为：

    ```text
    R1(A, B)
    R2(B, C)
    ```

    二者都在 BCNF 中，且该分解无损并保持依赖。

### BCNF Decomposition

BCNF 分解算法的思想很直接：只要某个子模式中存在违反 BCNF 的非平凡依赖 $\alpha\to\beta$，就把它拆成两部分：

```text
Ri1 = alpha union beta
Ri2 = Ri - beta
```

算法形式如下：

```text
result := {R}
compute F+
while some Ri in result is not in BCNF:
    choose a nontrivial dependency alpha -> beta on Ri
    such that alpha is not a superkey
    result := (result - Ri) union (alpha, beta) union (Ri - beta)
```

最终得到的每个子模式都在 BCNF 中，并且分解是无损连接的。

!!! warning "BCNF 的代价"
    BCNF 分解不一定保持依赖。也就是说，BCNF、无损连接、依赖保持这三个目标不能总是同时满足。

经典例子是：

```text
R = (J, K, L)
F = {JK -> L, L -> K}
```

其中候选键是 `JK` 和 `JL`。依赖 `L -> K` 违反 BCNF，因为 `L` 不是超键。但任何 BCNF 分解都会破坏 `JK -> L` 的依赖保持。

## Third Normal Form

第三范式（Third Normal Form, 3NF）比 BCNF 稍弱。引入它的动机是：有时 BCNF 分解不能保持依赖，但我们又希望更新检查尽量高效。

关系模式 $R$ 在 $F$ 下满足 3NF，当且仅当对 $F^+$ 中每个 $\alpha\to\beta$，至少满足以下条件之一：

1. $\alpha\to\beta$ 是平凡依赖；
2. $\alpha$ 是 $R$ 的超键；
3. $\beta-\alpha$ 中的每个属性都属于 $R$ 的某个候选键。

第三个条件是对 BCNF 的最小放松。它允许某些左部不是超键的依赖存在，但要求右部是主属性。

!!! note "BCNF 与 3NF"
    如果一个关系模式在 BCNF 中，那么它一定在 3NF 中。但反过来不一定成立。

### 3NF Decomposition

3NF 的一个重要优点是：总能得到一个无损连接、依赖保持的 3NF 分解。

算法如下：

```text
Fc := canonical cover of F
result := empty set
for each dependency alpha -> beta in Fc:
    if no schema in result contains alpha beta:
        add schema (alpha beta) into result
if no schema in result contains a candidate key for R:
    add any candidate key of R into result
return result
```

其中前半部分保证依赖保持，最后加入候选键是为了保证无损连接。

!!! example "3NF 分解"
    设：

    ```text
    Banker-info-schema = (
        branch-name, customer-name,
        banker-name, office-number
    )

    F = {
        banker-name -> branch-name office-number,
        customer-name branch-name -> banker-name
    }
    ```

    其候选键是 `{customer-name, branch-name}`。根据 3NF 分解算法，可以得到：

    ```text
    Banker-office-schema = (banker-name, branch-name, office-number)
    Banker-schema = (customer-name, branch-name, banker-name)
    ```

    第二个关系中已经包含原模式的候选键，所以不需要额外添加关系。

### BCNF vs 3NF

| 性质 | BCNF | 3NF |
| --- | --- | --- |
| 冗余控制 | 更强 | 稍弱 |
| 无损连接 | 总能做到 | 总能做到 |
| 依赖保持 | 不一定能做到 | 总能做到 |
| 使用场景 | 优先目标 | 当依赖保持很重要时使用 |

因此设计目标一般是先尝试 BCNF。如果无法同时保持依赖，就需要在“不保持依赖”和“接受 3NF 的少量冗余”之间做取舍。

??? tip "SQL 中的现实问题"
    SQL 对函数依赖的直接支持其实比较有限。除了主键、唯一约束这类“左部是键”的依赖之外，普通函数依赖通常需要用 assertion 或触发器等方式维护，代价较高。因此依赖保持在实际系统中仍然很有价值。

## Multivalued Dependencies

即使一个关系模式已经在 BCNF 中，也未必完全消除了冗余。这时需要讨论**多值依赖**（Multivalued Dependency, MVD）。

考虑关系：

```text
classes(course, teacher, book)
```

含义是某门课有若干可授课教师，也有若干指定教材。教师集合和教材集合对同一门课来说是彼此独立的：无论哪位老师教这门课，教材集合都相同。

如果直接存成三列表，就会出现组合冗余：

```text
(database, Avi, DB Concepts)
(database, Avi, DB system)
(database, Hank, DB Concepts)
(database, Hank, DB system)
```

如果新增一个老师 `Sara` 可以教 `database`，那么需要为她和每本教材都插入一行。这并不是函数依赖能解决的问题，因为这个表中通常只有平凡函数依赖，甚至已经满足 BCNF。

更好的设计是分解为：

```text
teaches(course, teacher)
text(course, book)
```

### Definition of MVD

设 $R$ 是关系模式，$\alpha\subseteq R,\beta\subseteq R$。如果对于任意合法关系 $r(R)$ 中满足 $t_1[\alpha]=t_2[\alpha]$ 的两个元组，总能在 $r$ 中找到元组 $t_3,t_4$，使得它们在 $\alpha$ 上相同，并且交换 $\beta$ 与剩余属性的组合后仍然存在，那么称：

$$
\alpha\twoheadrightarrow\beta
$$

在 $R$ 上成立。

直观地说，$\alpha\twoheadrightarrow\beta$ 表示：给定 $\alpha$ 后，$\beta$ 的取值集合与剩余属性的取值集合相互独立。

!!! example
    在 `classes(course, teacher, book)` 中：

    ```text
    course ->-> teacher
    course ->-> book
    ```

    因为对于一门课，教师集合和教材集合是两个独立的多值集合。

如果 $\beta\subseteq\alpha$，或者 $\alpha\cup\beta=R$，则该多值依赖是平凡的。并且每一个函数依赖都可以看成一种特殊的多值依赖。

## Fourth Normal Form

第四范式（Fourth Normal Form, 4NF）用来处理多值依赖带来的冗余。关系模式 $R$ 在函数依赖和多值依赖集合 $D$ 下满足 4NF，当且仅当对于 $D^+$ 中每个形如：

$$
\alpha\twoheadrightarrow\beta
$$

的多值依赖，至少满足：

1. 该多值依赖是平凡的；
2. $\alpha$ 是 $R$ 的超键。

!!! note
    如果一个关系在 4NF 中，那么它一定在 BCNF 中。因为函数依赖也是一种特殊的多值依赖。

4NF 分解算法与 BCNF 分解算法非常相似。遇到违反 4NF 的非平凡多值依赖 $\alpha\twoheadrightarrow\beta$，且 $\alpha$ 不是超键，就分解为：

```text
R1 = alpha union beta
R2 = R - beta
```

最终得到的各个子模式都在 4NF 中，并且分解是无损连接的。

!!! example "4NF 分解"
    设：

    ```text
    R = (A, B, C, G, H, I)
    D = {
        A ->-> B,
        B ->-> HI,
        CG ->-> H
    }
    ```

    因为 `A ->-> B` 非平凡，且 `A` 不是超键，所以 $R$ 不满足 4NF。可以先分解为：

    ```text
    R1(A, B)
    R2(A, C, G, H, I)
    ```

    若 `R2` 仍不满足 4NF，则继续根据其中的多值依赖分解，直到每个子模式都满足 4NF。

## Overall Design Process

最后，我们回到数据库设计的整体视角。规范化理论通常假设某个关系模式 $R$ 已经给定。这个 $R$ 可能来自：

- ER 图转换得到的一组表；
- 包含所有属性的泛关系（universal relation）；
- 一次临时或经验性的关系设计。

如果 ER 图设计得足够好，并且实体、联系都识别准确，那么由 ER 图转换得到的表一般不需要再做大量规范化。但现实中设计很容易不完美，比如把 `department-number` 和 `department-address` 都放进 `employee` 实体中，此时会出现：

```text
department-number -> department-address
```

这说明 `department` 其实应该被建成单独的实体集。

### Denormalization

规范化不是绝对目标。有时为了性能，我们会故意使用非规范化设计。

比如展示：

```text
(customer-name, account-number, balance)
```

可能需要连接 `account` 和 `depositor`。如果这个查询非常频繁，可以考虑：

- 建一个冗余关系，直接保存这些属性；
- 建一个物化视图，由数据库维护连接结果。

!!! warning "反规范化的代价"
    反规范化可以加速查询，但会带来额外存储、额外更新成本，以及维护一致性的风险。它应该是性能分析之后的工程选择，而不是默认设计方式。

### Other Design Issues

有些坏设计并不能单纯靠范式发现。比如要记录公司每年的收益：

=== "坏设计 1"
    每年一张表：

    ```text
    earnings-2024(company-id, earnings)
    earnings-2025(company-id, earnings)
    ```

    这种设计跨年份查询困难，而且每年都要建新表。

=== "坏设计 2"
    每年一个属性：

    ```text
    company-year(company-id, earnings-2024, earnings-2025, ...)
    ```

    这类似电子表格中的交叉表，扩展年份时需要改模式。

=== "更好的设计"
    把年份作为普通属性：

    ```text
    earnings(company-id, year, amount)
    ```

    这种设计更符合关系模型，也更方便查询和扩展。

## Summary

本章的主线可以总结为：

1. 1NF 要求属性域原子化；
2. 坏的关系模式会导致冗余和各种更新异常；
3. 函数依赖是判断关系模式好坏的核心工具；
4. 属性闭包可以用于判断超键、候选键和函数依赖；
5. 正则覆盖用于简化函数依赖集；
6. 分解需要尽量保证无损连接和依赖保持；
7. BCNF 更严格，但可能不保持依赖；
8. 3NF 稍弱，但总能做到无损连接和依赖保持；
9. 多值依赖揭示 BCNF 仍可能存在的冗余；
10. 4NF 用来进一步处理独立多值集合带来的组合冗余。

!!! abstract "范式之间的关系"
    一般可以理解为：

    ```text
    4NF => BCNF => 3NF => 1NF
    ```

    范式越高，冗余约束越强；但实际设计中还要综合考虑依赖保持、查询性能和维护成本。
