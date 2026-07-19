# JavaSE 学习清单

> 共 8 篇笔记，按学习顺序排列。勾选即完成，可同时当做目录使用。

---

## 一、走进Java语言

- [x] **[[JavaSE 核心内容 - JavaSE 笔记（一）走进Java语言|📖 查看笔记全文]]**

### 1.1 计算机思维导论

- [x] 计算机的世界 —— 冯诺依曼结构、CPU/内存/硬盘分工
- [x] 操作系统概述 —— 内核/系统调用/用户态与内核态
- [x] 计算机编程语言 —— 机器语言→汇编→高级语言演进
- [x] 走进Java语言 —— Java 跨平台原理（JVM/字节码）、JDK/JRE/JVM 区别

### 1.2 环境安装与IDE使用

- [x] JDK下载与安装 —— 环境变量（JAVA_HOME、Path）配置
- [x] IDEA安装与使用 —— 创建工程、快捷键、运行/调试

---

## 二、面向过程编程

- [x] **[[JavaSE 核心内容 - JavaSE 笔记（二）面向过程编程|📖 查看笔记全文]]**

### 2.1 Java程序基础

- [x] 程序代码基本结构 —— 类→main方法→语句
- [x] 注释 —— 单行/多行/文档注释（`/** */`）
- [x] 变量与常量 —— 标识符规则、`final` 修饰常量

### 2.2 基本数据类型

- [x] 计算机中的二进制表示 —— 原码、反码、**补码**（负数的存储方式）
- [x] 老鼠试药问题 —— 二进制枚举思想
- [x] 整数类形 —— byte/short/int/long 范围、隐式类型转换、溢出导致的值突变
- [x] 浮点类型 —— IEEE 754 公式 $V=(-1)^S\times M\times 2^E$、精度丢失（0.1+0.2≠0.3）
- [x] 字符类型 —— ASCII→GB2312→GBK→Unicode→UTF-8/UTF-16 编码演进
- [x] 布尔类型 —— `true`/`false`，只有两种值
- [x] (Java 10) 局部变量类型推断 —— `var` 关键字，编译器自动推断类型

### 2.3 运算符

- [x] 赋值运算符 —— `=` 及复合赋值 `+=`、`-=`、`*=`、`/=`
- [x] 算术运算符 —— 加减乘除取模、字符串拼接（`+`）、优先级
- [x] 括号运算符 —— 提升优先级、**强制类型转换**（`(int)`、截断与精度损失）
- [x] 自增自减运算符 —— 前置 `++a` vs 后置 `a++` 的区别
- [x] 位运算符 —— 与`&`、或`|`、异或`^`、取反`~`、左移`<<`、右移`>>`（算数右移）、无符号右移`>>>`
- [x] 关系运算符 —— `>`、`<`、`>=`、`<=`、`==`、`!=`
- [x] 逻辑运算符 —— 与`&&`、或`||`、非`!`，**短路特性**

### 2.4 流程控制

- [x] 代码块与作用域 —— `{}` 划定的变量生命周期
- [x] 选择结构 —— `if`/`else if`/`else`，嵌套判断
- [x] (Java 14) Switch表达式 —— 箭头语法、`yield`、模式匹配
- [x] 循环结构 —— `while`/`do...while`/`for`、`break`/`continue`、嵌套循环
- [x] (Java 9) 交互式编程 —— `jshell` 工具

### 2.5 实战练习

- [x] 寻找水仙花数 —— 100~999 中各位数字立方和等于本身的数
- [x] 打印九九乘法表 —— 嵌套 `for` 循环
- [x] 斐波那契数列 —— 迭代 / 递归两种方式

---

## 三、面向对象基础

- [ ] **[[JavaSE 核心内容 - JavaSE 笔记（三）面向对象基础|📖 查看笔记全文]]**

### 3.1 类与对象

- [ ] 类的定义与对象创建 —— `class` 关键字、`new` 实例化、内存分配
- [ ] 对象的使用 —— 属性访问、方法调用
- [ ] 方法创建与使用 —— 返回值、参数列表、`return`
- [ ] 方法进阶使用 —— **方法重载**（同名不同参）、可变长参数 `(int... nums)`
- [ ] 构造方法 —— `new` 时自动调用、无参/有参、`this()` 调用链
- [ ] 静态变量和静态方法 —— `static` 修饰，属于类而非对象，`static{}` 代码块初始化

### 3.2 包和访问控制

- [ ] 包声明和导入 —— `package`、`import`、命名规范（域名倒写）
- [ ] 访问权限控制 —— `private`/默认/`protected`/`public` 四层 | 同类/同包/子类/全局

### 3.3 封装、继承和多态

- [ ] 类的封装 —— `getter`/`setter`、属性私有化
- [ ] 类的继承 —— `extends`、`Object` 为默认父类、`super` 调用父类、`final` 禁止继承
- [ ] 顶层Object类 —— 所有类的"老祖宗"，`toString()`/`equals()`/`hashCode()`
- [ ] (Java 16) 类型判断模式匹配 —— `instanceof` 自动转换 `if (obj instanceof String s)`
- [ ] 方法的重写 —— `@Override`、子类覆盖父类方法、`final` 禁止重写
- [ ] **抽象类** —— `abstract` 修饰、不能实例化、子类必须实现抽象方法
- [ ] **接口** —— `interface`、多实现（`implements`）、`default` 方法
- [ ] (Java 8) 接口默认和静态方法 —— `default` 和 `static` 方法
- [ ] (Java 9) 接口中的private方法 —— 抽取 `default` 的公共逻辑

### 3.4 其他类型

- [ ] 枚举类 —— `enum` 关键字、单例、`values()`/`valueOf()`
- [ ] (Java 16) 记录类型 —— `record`，不可变数据载体，自动生成 `equals`/`hashCode`/`toString`
- [ ] (Java 17) 密封类型 —— `sealed`，限制哪些类可以继承/实现

---

## 四、面向对象高级篇

- [ ] **[[JavaSE 核心内容 - JavaSE 笔记（四）面向对象高级篇|📖 查看笔记全文]]**

### 4.1 基本类型包装类

- [ ] 包装类介绍 —— int→Integer、自动装箱/拆箱、缓存池（-128~127）
- [ ] 特殊包装类 —— BigInteger/BigDecimal 大数运算

### 4.2 数组

- [ ] 一维数组 —— 声明 `int[] arr`、初始化、遍历、`length`
- [ ] 多维数组 —— 二维数组 `int[][]`，本质是"数组的数组"
- [ ] 可变长参数 —— `(int... nums)` 本质是数组

### 4.3 字符串

- [ ] String类 —— **不可变性**、常量池、`intern()`、常用 API（`indexOf`/`substring`/`split`/`replace`）
- [ ] StringBuilder类 —— 可变字符串、线程不安全、性能优于 String 拼接
- [ ] (Java 11) 字符串增强 —— `strip()`/`isBlank()`/`lines()`/`repeat()`
- [ ] (Java 15) 文本块 —— `"""` 三段引号，保留缩进和换行
- [ ] 正则表达式 —— `Pattern`/`Matcher`、元字符、分组
- [ ] (Java 21/22/25) switch模式匹配 —— `switch` 中解构类型和值

### 4.4 内部类

- [ ] 成员内部类 —— 类中定义类，可访问外部类成员
- [ ] 静态内部类 —— `static` 修饰的内部类
- [ ] 局部内部类 —— 方法中定义的类
- [ ] 匿名内部类 —— `new Interface() { 实现 }`，常用于回调
- [ ] **Lambda表达式** —— `(参数) -> { 代码体 }`，函数式接口的简化写法
- [ ] 方法引用 —— `类名::静态方法`、`对象::实例方法`、`类名::实例方法`

### 4.5 异常机制

- [ ] 异常的类型 —— Error/Exception、**检查异常** vs 非检查异常、继承树
- [ ] 自定义异常 —— 继承 `Exception` 或 `RuntimeException`
- [ ] 抛出异常 —— `throw` 抛异常对象、`throws` 声明可能抛出
- [ ] 异常的处理 —— **`try-catch-finally`**、多重 `catch`、`try-with-resources`
- [ ] 断言表达式 —— `assert` 调试验证

### 4.6 常用工具类介绍

- [ ] 数学工具类 —— `Math` 常用方法（`abs`/`pow`/`sqrt`/`random`）
- [ ] 数组工具类 —— `Arrays.sort()`/`binarySearch()`/`toString()`/`copyOf()`
- [ ] 日期相关类 —— `Date`、`SimpleDateFormat`（格式化）
- [ ] (Java 8) 新的日期类 —— `LocalDate`/`LocalTime`/`LocalDateTime`、`DateTimeFormatter`

### 4.7 实战练习

- [ ] 冒泡排序算法 —— 相邻比较交换，时间复杂度 $O(n^2)$
- [ ] 二分搜索算法 —— 有序数组中对半查找，$O(\log n)$
- [ ] 青蛙跳台阶问题 —— 递归（斐波那契本质）+ 动态规划优化
- [ ] 回文串判断 —— 双指针法
- [ ] 汉诺塔求解 —— 递归经典，$2^n - 1$ 步

---

## 五、泛型程序设计

- [ ] **[[JavaSE 核心内容 - JavaSE 笔记（五）泛型程序设计|📖 查看笔记全文]]**

### 5.1 泛型

- [ ] 泛型类 —— `<T>` 类型参数，编译期类型检查
- [ ] 泛型与多态 —— 泛型不支持多态参数传递（`List<Object>` 不能接收 `List<String>`）
- [ ] 泛型方法 —— `<T> T method(T t)`，类型由实参推断
- [ ] 泛型的界限 —— `<? extends T>` 上界、`<? super T>` 下界
- [ ] **类型擦除** —— 编译后泛型信息消失，字节码中还原为 Object + 强制转换
- [ ] 协变和逆变 —— 生产者 `extends`（读）、消费者 `super`（写），PECS 原则

### 5.2 (Java 8) 函数式编程

- [ ] 函数式接口 —— `@FunctionalInterface`，只有一个抽象方法
- [ ] (Java 8) 判空包装 —— `Optional<T>`，`map`/`flatMap`/`orElse` 链式处理
- [ ] (Java 9/10) 判空包装增强 —— `or()`、`ifPresentOrElse()`、`orElseThrow()`

### 5.3 数据结构基础

- [ ] 线性表：**顺序表** —— `Object[]` 实现、增删查改、扩容机制
- [ ] 线性表：**链表** —— 节点 `Node`、头插/尾插/删除、双向链表
- [ ] 线性表：**栈** —— FILO、`push`/`pop`/`peek`、括号匹配应用
- [ ] 线性表：**队列** —— FIFO、`offer`/`poll`/`peek`、BFS 应用
- [ ] 树：**二叉树** —— 节点结构、前/中/后序遍历、层序遍历
- [ ] 树：**二叉查找树和平衡二叉树** —— BST 左小右大、AVL 旋转
- [ ] 树：**红黑树** —— 5条性质、自平衡、`TreeMap` 底层
- [ ] **哈希表** —— 哈希函数、冲突解决（链地址法）、负载因子

### 5.4 实战练习

- [ ] 反转链表 —— 迭代法（三指针）+ 递归法
- [ ] 括号匹配问题 —— 栈的应用
- [ ] 实现计算器 —— 中缀表达式转后缀，栈计算

---

## 六、集合类与IO

- [ ] **[[JavaSE 核心内容 - JavaSE 笔记（六）集合类与IO|📖 查看笔记全文]]**

### 6.1 集合类

- [ ] 集合根接口 —— `Collection` 与 `Map` 两大体系、继承关系图
- [ ] **List列表** —— `ArrayList`（数组实现，查询快）vs `LinkedList`（链表实现，增删快）
- [ ] 迭代器 —— `Iterator`、`for-each` 语法糖、`fail-fast` 机制
- [ ] Queue和Deque —— 队列/双端队列，`ArrayDeque`、`PriorityQueue`
- [ ] **Set集合** —— `HashSet`（哈希无序）vs `TreeSet`（红黑树有序）vs `LinkedHashSet`（保持插入顺序）
- [ ] **Map映射** —— `HashMap`（**数组+链表+红黑树**、扩容机制）vs  `TreeMap` vs `LinkedHashMap`
- [ ] 比较相关接口 —— `Comparable`（自然排序）vs `Comparator`（定制排序）
- [ ] Collections工具类 —— `sort`/`reverse`/`shuffle`/`synchronized` 包装
- [ ] (Java 9) 集合工厂方法 —— `List.of()`/`Set.of()`/`Map.of()` 不可变集合
- [ ] (Java 21) 有序集合功能规范 —— `SequencedCollection`、`getFirst()`/`getLast()`
- [ ] (Java 8) **Stream流** —— 中间操作（`filter`/`map`/`sorted`）+ 终端操作（`collect`/`reduce`/`forEach`）
- [ ] (Java 9/11/16) Stream增强方法 —— `takeWhile`/`dropWhile`、`toList()`
- [ ] (Java 24) 流聚集器 —— `Stream Gatherer`

### 6.2 Java I/O

- [ ] 文件字节流 —— `FileInputStream`/`FileOutputStream`，读写原始字节
- [ ] 文件字符流 —— `FileReader`/`FileWriter`，处理文本文件
- [ ] (Java 7/8/11) 文件工具类 —— `Files`/`Path`/`Paths`，`readString`/`writeString`
- [ ] 缓冲流 —— `BufferedInputStream`/`BufferedReader`，减少磁盘IO次数
- [ ] 转换流 —— `InputStreamReader`/`OutputStreamWriter`，字节↔字符桥接
- [ ] 打印流 —— `PrintStream`（`System.out`）/`PrintWriter`
- [ ] 数据流 —— `DataInputStream`/`DataOutputStream`，读写基本类型
- [ ] 对象流 —— `ObjectInputStream`/`ObjectOutputStream`、**序列化**`Serializable`
- [ ] 字节数组流 —— `ByteArrayInputStream`/`ByteArrayOutputStream`，内存中操作
- [ ] (Java 9/11/12) 输入流快捷操作 —— `readAllBytes()`、`transferTo`

### 6.3 实战：图书管理系统

- [ ] 图书管理系统 —— CRUD + 文件持久化，综合运用集合与IO

---

## 七、多线程与反射

- [ ] **[[JavaSE 核心内容 - JavaSE 笔记（七）多线程与反射|📖 查看笔记全文]]**

### 7.1 多线程

- [ ] 线程的创建和启动 —— `extends Thread` vs `implements Runnable`、`start()` vs `run()`
- [ ] 线程的休眠和中断 —— `sleep()`、`interrupt()`、`isInterrupted()`
- [ ] 线程的优先级 —— `setPriority(1~10)`，只是"建议"
- [ ] 线程的礼让和加入 —— `yield()` 让出CPU、`join()` 等待线程结束
- [ ] 线程锁和线程同步 —— **`synchronized`**、`wait()`/`notify()`、`Lock`/`ReentrantLock`
- [ ] **死锁** —— 产生条件（互斥/持有并等待/不可剥夺/循环等待）、排查方法
- [ ] wait和notify方法 —— 生产消费模型的基础，必须在 `synchronized` 块内
- [ ] ThreadLocal的使用 —— 每个线程独立的变量副本
- [ ] 定时器 —— `Timer`、`ScheduledExecutorService`、`@Scheduled`
- [ ] 守护线程 —— `setDaemon(true)`，主线程结束则自动销毁
- [ ] 再谈集合类 —— `ConcurrentHashMap`、`CopyOnWriteArrayList`、`BlockingQueue`
- [ ] (Java 8) 并行流 —— `parallelStream()`，ForkJoinPool 底层
- [ ] 实战：生产者与消费者 —— `wait()`/`notify()` 经典实现
- [ ] (Java 21) 线程生成器 —— `Thread.Builder` 线程工厂
- [ ] (Java 21) **虚拟线程** —— `Thread.startVirtualThread()`，轻量级协程

### 7.2 反射

- [ ] Java类加载机制 —— 双亲委派模型、Bootstrap/Extension/Application
- [ ] Class类获取 —— `类名.class`/`obj.getClass()`/`Class.forName()`
- [ ] Class对象与多态 —— 反射中的类型处理
- [ ] 创建类对象 —— `Constructor.newInstance()`，处理私有构造
- [ ] 调用类方法 —— `Method.invoke()`，包括私有方法
- [ ] 修改类的属性 —— `Field.set()`，包括私有属性
- [ ] (Java 9) 模块化机制 —— `module-info.java`、`exports`、`requires`
- [ ] 类加载器 —— 自定义类加载器、`findClass`/`loadClass`

### 7.3 注解

- [ ] 预设注解 —— `@Override`/`@Deprecated`/`@SuppressWarnings`
- [ ] 元注解 —— `@Target`/`@Retention`/`@Documented`/`@Inherited`/`@Repeatable`
- [ ] 注解的使用 —— 自定义注解、注解属性定义
- [ ] 反射获取注解 —— `getAnnotation()`/`isAnnotationPresent()`

---

## 八、GUI程序开发

- [ ] **[[JavaSE 核心内容 - JavaSE 笔记（八）GUI程序开发|📖 查看笔记全文]]**

### 8.1 AWT组件介绍

- [ ] 基本框架 —— `Frame`/`Panel` 窗口容器、`paint()` 重绘
- [ ] 监听器 —— 点击/键盘/鼠标等事件绑定
- [ ] 常用组件 —— Button/Label/TextField/TextArea/CheckBox/Choice
- [ ] 布局和面板 —— `FlowLayout`/`BorderLayout`/`GridLayout`/`CardLayout`
- [ ] 滚动面板和列表 —— `ScrollPane`/`List`
- [ ] 菜单栏 —— `MenuBar`/`Menu`/`MenuItem`/`PopupMenu`
- [ ] 对话框 —— `Dialog`、`FileDialog`（文件选择）
- [ ] 自定义组件 —— 重写 `paint()` 绘制自定义图形
- [ ] 窗口修饰和自定义形状 —— 窗口样式定制

### 8.2 Swing组件介绍

- [ ] 基本使用 —— `JFrame`/`JPanel`、Swing 线程模型（EDT）
- [ ] 新增组件介绍 —— `JTable`/`JTree`/`JProgressBar`/`JFileChooser`
- [ ] 多面板和分割面板 —— `JTabbedPane`/`JSplitPane`
- [ ] 选项窗口 —— `JOptionPane` 标准对话框、`JColorChooser` 颜色选择
- [ ] 自定义主题 —— `LookAndFeel`（LF）切换

### 8.3 项目实战

- [ ] Intellij IDEA Extreme —— 简易 IDEA 模拟项目

---

## 学习进度统计

| 篇章 | 小节数 | 已完成 | 进度 |
|:-----|:------:|:------:|:----:|
| 一、走进Java语言 | 6 | 0 | 0% |
| 二、面向过程编程 | 28 | 0 | 0% |
| 三、面向对象基础 | 21 | 0 | 0% |
| 四、面向对象高级篇 | 28 | 0 | 0% |
| 五、泛型程序设计 | 18 | 0 | 0% |
| 六、集合类与IO | 27 | 0 | 0% |
| 七、多线程与反射 | 30 | 0 | 0% |
| 八、GUI程序开发 | 14 | 0 | 0% |
| **总计** | **172** | **0** | **0%** |
