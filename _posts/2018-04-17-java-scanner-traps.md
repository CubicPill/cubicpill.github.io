---
layout: post
title: Java 中 Scanner 使用的一些坑
key: 2018-04-17-java-scanner-traps

---       
本文是为 2018 春季 Java A 课程的同学们所写的教程, 会结合一些在课堂中碰到的案例进行说明.    
在 Java 中, 我们经常使用 ```Scanner``` 类来获取用户输入, 但是有些时候会遇到一些奇怪的问题. 本文整理了一下在批改作业中遇到的例子和同学们的疑问并做出了解释, 希望可以帮助同学们加深对 Scanner 背后机制的理解.    
<!--more-->
## 阅读此文前, 你需要:
1. 了解 Scanner 的基本用法, 清楚 ```nextLine()```, ```nextInt()``` 等函数的用法与区别.    
2. 了解有关 ```stdin``` 和 ```stdout``` 的[基本概念](https://en.wikipedia.org/wiki/Standard_streams), 以及对[输入输出重定向](https://en.wikipedia.org/wiki/Redirection_(computing))的基本了解.    
3. 善用搜索引擎查找相关知识.

## 坑1: 为什么我的输入只有行内开始的一个数, 后面的数都不见了？
### 示例
下面的代码中, ```testInputMultiple()``` 是我们从用户输入获取两个整数的常用做法. 而```testInputMultipleWrong()``` 函数是在批改作业过程中发现的一些写法. 这种写法可能在某些情况下不会出现问题 (如果每输入一个数字就按下回车的话), 但在同一行内连续按空格分割输入的时候, 就会出现一些问题.
```java
import java.util.Scanner;
public class ScannerTrap1{
    
    public static void testInputMultiple(){
        System.out.println("Input:");
        Scanner s =new Scanner(System.in);
        
        int a=s.nextInt();
        int b=s.nextInt();
        
        System.out.printf("I get: %d, %d\n",a,b);
    }
    public static void testInputMultipleWrong(){
        System.out.println("Input:");
        Scanner s1 =new Scanner(System.in);
        Scanner s2 =new Scanner(System.in);
        int a=s1.nextInt();
        int b=s2.nextInt();
        
        System.out.printf("I get: %d, %d\n",a,b);
    }
    public static void main(String args[]){
        testInputMultiple();
        testInputMultipleWrong();
    }
}
```
运行, 输入一些数据测试, 会发现在执行到第二个函数时, 即使输入了数字, 程序也不会结束, 直到我们另外输入一个数字并按回车. 但第二个数字竟然不是第一行我们输入的数字, **why?**    
切换成重定向测试一下, 这次程序直接抛出了 ```NoSuchElementException```.    

![](\content\images\2018\java_scanner\console_trap1.png)

### 原因探究
观察两个函数的差别, 仅仅是因为第二个函数重复创建了多个 Scanner ,导致了错误的产生. 原因在于 Scanner 的 ```nextInt()``` 函数被调用时, 会从标准输入的缓冲区内读取数据并存入自己的存储区域, 然后提取出这一行中的第一个整数, 并作为函数的返回值返回.     
下面的图表示了我们使用 Scanner 的一般过程: ```Scanner``` 对象从 ```System.in``` 流中读取数据, 并放入自己的缓冲区, 等待被其他代码调用返回.    

![](\content\images\2018\java_scanner\scanner_fig1.png)

 在上面的示例中, 由于我们使用了两个 ```Scanner``` 对象读取数据, ```1 3``` 一行实际上被 ```s1``` 全部取走, 并留在 ```s1``` 的缓冲区内. 因此在接受用户输入的时候, ```4``` 的一行才是真正被 ```s2``` 所接收到的数据. 在重定向之时, 因为输入在两行之后已经结束, 因此 ```s2``` 的 ```nextInt()``` 函数无法读取任何数据, 就会产生 ```NoSuchElementException``` 异常.    

![](\content\images\2018\java_scanner\scanner_fig2.png)

### 解决方案
只用一个 Scanner 对象接收标准输入, 不要重复创建.    
## 坑2: 为什么只关闭了一个 Scanner, 其余的 Scanner 都不能用了?
### 示例
下面的代码中, 我们创建了两个 ```Scanner``` 对象. 其中 ```s1``` 在接收到一个整数之后, 我们将其关闭. 按照一般的经验来说, s2 应当不受影响. 但是当我们运行这段代码时, 却出现了异常:    
```java
import java.util.Scanner;
public class ScannerTrap2{
    public static void main(String args[]){
        int n=0;
        Scanner s1=new Scanner(System.in);
        Scanner s2=new Scanner(System.in);
        System.out.println("Input for s1:");
        n=s1.nextInt();
        System.out.printf("Echo: %d\n",n);
        s1.close();
        System.out.println("Input for s2:");
        n=s2.nextInt();
        System.out.printf("Echo: %d\n",n);
        s2.close();
    }
}
```

![](\content\images\2018\java_scanner\console_trap2.png)

抛出了 ```NoSuchElementException```. 但是我们还没有输入, ```s2``` 也未被关闭, 为什么会出现这种情况呢?    
### 原因探究    

通常我们在使用完输入流这一类资源之后, 为了释放资源供其他程序使用, 需要调用 ```close()``` 函数. 但是, 在调用 Scanner 的 ```close()``` 函数时, Scanner 也会自动调用它所读取的输入流的 ```close()``` 函数. 也就是说, ```System.in``` 流会随着 ```s1``` 的关闭而被关闭. 下图演示了这一过程: s1 被关闭, 在关闭的过程中也调用了 ```System.in``` 的关闭函数.    

![](\content\images\2018\java_scanner\scanner_fig3.png)

所以, 当我们调用 ```s2``` 的 ```nextInt()``` 方法时, 由于 ```System.in``` 早已被关闭, 因此 ```s2``` 无法读取到任何用户的输入数据, 而是直接抛出异常:    

![](\content\images\2018\java_scanner\scanner_fig4.png)

### 解决方案
在确认完全不会使用 Scanner 所用的输入流之后, 再关闭 Scanner.    
## 坑3: 为什么没有关闭 Scanner, 但新的 Scanner 也会报错?
### 示例
在下面的代码中, 我们从重定向的文件内接收输入. 文件内有两行, 分别对应着两次 ```nextLine()``` 的输入. 
```java
import java.util.Scanner;
public class ScannerTrap3{
    
    public static void testInput(){
        System.out.println("Creating Scanner......");
        Scanner s =new Scanner(System.in);
        System.out.println("I'm created! Ready to get input...");
        String str=s.nextLine();
        System.out.println("I'm OK! Get:");
        System.out.println(str);
    }
    public static void main(String args[]){
        testInput();
        testInput();
    }
}
```
运行代码, 第一次对 ```testInput()``` 的调用看起来没有问题, 但第二次的调用却又抛出了 ```NoSuchElementException``` 异常.

![](\content\images\2018\java_scanner\console_trap3.png)

### 原因探究
产生这种错误的原因和第一种情况很相似, 但又有些许的不同.     

我们知道, 标准输入不仅仅可以接受用户键盘的输入, 还可以通过重定向的方式, 将文件或其他进程的输出作为程序的标准输入. 如图所示:    

![](\content\images\2018\java_scanner\scanner_fig5.png)

但是文件重定向输入和用户键盘输入的一点不同是, 用户的输入是在每次按下回车键时, 发送到输入缓冲区的. 因此, 可以认为用户的输入是输入一行, 程序读取并处理后再输入第二行 (因为一般情况下, 用户的输入间隔时间远远大于处理所需要的时间). 但文件有所不同, 文件是以文件结尾 (End Of File, EOF) 作为输入终止的标志. 可以认为, 文件虽然可能有很多行, 但这些多行会一起被输入到缓冲区, 并被 Scanner 读取到自己的缓冲区. 当文件被完全输入到缓冲区, 并且缓冲区被完全读取完毕的时候, 标准输入流会被自动关闭. 而在接受用户输入时, 只要用户不主动按下组合键 (Ctrl+D) 发送 EOF 标记, 标准输入流将一直保持打开状态.  

![](\content\images\2018\java_scanner\scanner_fig6.png)

所以, 问题的原因到此就显而易见了: 在采用文件重定向输入时, 由于每次进入函数都会创建新的 Scanner 对象, 因此第一次创建的 Scanner 对象将缓冲区的数据全部读取到自己的缓冲区内. 缓冲区此时已空, 因此标准输入流被自动关闭. 函数第二次被执行时, 由于 ```System.in``` 已被关闭, 因此抛出了 ```NoSuchElementException``` 异常.

![](\content\images\2018\java_scanner\scanner_fig7.png)
### 解决方案
由于 OJ 采用的测试方法是将测试用例文件重定向到标准输入, 因此最好全局使用同一个 ```Scanner``` 对象, 不要重复创建. 可以将 Scanner 对象声明为类的一个 Field, 就可以实现在类的函数中的通用.