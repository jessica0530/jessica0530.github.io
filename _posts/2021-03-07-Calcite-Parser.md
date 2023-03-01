---
layout: post
title: Calcite Parser
categories: [calcite]
description: calcite
keywords: calcite
---

## 编译知识

词法分析(lexing): 词法分析就是将文本分解成Token,Token就是具有特殊含义的原子单位,如语言的保留字,标点符号,数字,字符串等

语法分析(paring):语法分析器使用词法分析从输入中分离出一个个的token，并将token流作为其输入

根据某种给定的形式文法对由输入的token进行分析并确定其语法结构的过程

自顶向下分析,对应LL分析器

自底向上分析,对应LR分析器

例子:

```java
输入: A B C
文法: A->1;B->2;C->3

词法分析：得到 token{A,B,C}
语法分析: 得到结果 123
```



## javaCC 

使用递归下降语法解析,LL(k)

第一个L表示从左(left) 到右扫描输入

第二个L表示每次都进行最左推导(在推导语法树的过程中每次都替换句型中最左的非终结符为终结符)

最左推导

  如文法:

​    S ---> AB 

​    A ---> a

​    B ---> +CD

​    C  ----> a

​    D -----> a

 最左推导: S ---> AB ----> aB ----> a + CD ----> a+ aD -----> a+ aa

 最右推导: S ---->AB ---> A+CD-----> A+Ca ---->A+aa ----> a +aa



k表示每次向前探索(lookhead) k个终结符,以消除二义性

二义性 需要看多个token才能决策,只看一个token无法决定

LOOKHEAD:设置在解析过程中面临choice point 可以look head 的token 数量,缺省的值是1

调大K可以消除二义性,但会减慢解析速度

比如LOOKHEAD(2) 就表示要一次看两个token





## Javacc 

描述文件 基本结构

```java
options{
 JavaCC的选项
}

PARSER_BEGIN(解析器类名)
package 包名；
import 库名:
public class 解析器类名 {
   任意的Java代码
}
PARSER_END(解析器类名)

词法描述器

语法分析器
```

Option块 和class声明块

```java
options {
   static = false

}
PARSER_BEGIN(adder)
class Adder {
  class Adder {
     public static void main(String[] args) throws ParseException,TokenMgrError {
     Adder parser = new Adder(System.in);
     parser.start()
     }
  
  }
}
PARSER_END
```

STATIC 默认是 true,这里要将其修改为false,使得生成的函数不是static的。

PARSER_BEGIN(XXX) ..... PARSER_END(xxx)块,这里定义了一个名为Adder的类

## 词法描述器

```sql
//忽略的字符
SKIP ：{
 " "
 | "\t"
 |"\n"
 |"\r"
 |"\r\n"
}

//关键字
TOKEN：{
<PLUS : "+" >
|<NUMBER :(["0" - "9"]) +>
}
```

SKIP,表示将会被词法分析器忽略的部分

定义了一个名为PLUS 的token，用它来表示加号 +

定义了一个名为NUMBET的token,用它来表示(["0" -"9"]) +,即所有的正整数序列,注意（["0" -"9"]）+是一个正则表达式

## 语法分析器

描述由BNF生产式构成

```sql
//没有任何操作,只是约束了输入,如果输入不是 正整数+ 正整数,会报错（例子）
void Start():
{

}
{
<NUMBER>
(
<PLUS>
<NUMBER>
)
<EOF>

}
```



完整的adder.jj文件

```java
options {
STATIC = false;
}
PARSER_BEGIN(Adder)
class Adder {
   public static void main(String[] args) throws ParseException {
     Adder parser = new Adder(System.in)
       parser.start();
   }

}
PARSER_END(Adder)
  
SKIP:{" "|"\n" |"\r"| "\r\n"}
TOKEN:{<PLUS : "+">}
TOKEN:{<NUMBER:(["0"-"9"])}

void Start():
{}
{
  <NUMBER>
    ( <PLUS>
      <NUMBER>
    )*
    <EOF>
}
```

Adder是语法分析器

TokenMgrError是一个简单的定义错误的类,它是Throwable类的子类,用于定义在词法分析阶段检测到的错误

ParseException 是另一个定义错误的类。它是Exception和Throwable的子类，用于定义在语法分析阶段检测到的错误

Token类是一个用于表示token的类,我们在.jj文件中定义的每一个token(PLUS ,NUMBER, or EOF),在TOKEN 类中都有对应的一个整数属性来表示,此外每一个token都有名为image的string 类型的属性,用来表示token所代表的从输入中获取到的真实值

SimpleCharStream 是一个转接器类,用于把字符传递给语法分析器

AdderConstants 是一个接口,里面定义了一些词法分析器和语法分析器中都会用到的常量

AdderTokenManager 是词法分析器

## 语法介绍

java代码块{} 声明

```java
void javaCodeDemo();

{}

{
  {
    int i=0;
    System.out.println(i);
  }


}
```

java函数需要使用JAVACODE声明

```
JAVACODE void print(Token t) {
 System.out.println(t)
}
```

```java
void ifElseExpr();
{}
{
   (
   <SELECT> {System.out.println("if else select")}
     <UPDATE> {System.out.println("if else update")}
     <DELETE> {System.out.println("if else delete")}
     {
       System.out.println("other")
       
     }
     
     
   )
  
  
}
```

