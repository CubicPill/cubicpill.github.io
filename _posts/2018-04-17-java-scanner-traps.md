---
layout: post
title: Java 中 Scanner 使用的一些坑
key: 2018-04-17-java-scanner-traps

---       
本文是为 2018 春季 Java A 课程的同学们所写的教程, 会结合一些在课堂中碰到的案例进行说明.    
在 Java 中, 我们经常使用 ```Scanner``` 类来获取用户输入, 但是有些时候会遇到一些奇怪的问题. 本文整理了一下在批改作业中遇到的例子和同学们的疑问, 希望可以帮助同学们加深对 Scanner 背后机制的理解.    
<!--more-->
## 坑1: 为什么我的输入只有行内开始的一个数, 后面的数都不见了？
下面的代码中, ```testInputMultiple()``` 是我们从用户输入获取两个整数的常用做法. 而```testInputMultipleWrong()``` 函数是在批改作业过程中发现的一些同学的写法. 这种写法可能在某些情况下不会出现问题 (如果每输入一个数字就按下回车的话), 但在同一行内连续按空格分割输入的时候, 就会出现一些问题.
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

观察两个函数的差别, 仅仅是因为第二个函数重复创建了多个 Scanner ,导致了错误的产生. 原因在于 Scanner 的 ```nextInt()``` 函数被调用时, 会从标准输入的缓冲区内读取数据并存入自己的存储区域, 然后提取出这一行中的第一个整数, 并作为函数的返回值返回.     
下面的图表示了我们使用 Scanner 的一般过程: ```Scanner``` 对象从 ```System.in``` 流中读取数据, 并放入自己的缓冲区, 等待被其他代码调用返回.    
![](\content\images\2018\java_scanner\scanner_fig1.png)

 在上面的示例中, 由于我们使用了两个 ```Scanner``` 对象读取数据, ```1 3``` 一行实际上被 ```s1``` 全部取走, 并留在 ```s1``` 的缓冲区内. 因此在接受用户输入的时候, ```4``` 的一行才是真正被 ```s2``` 所接收到的数据. 在重定向之时, 因为输入在两行之后已经结束, 因此 ```s2``` 的 ```nextInt()``` 函数无法读取任何数据, 就会产生 ```NoSuchElementException``` 异常.    

![](\content\images\2018\java_scanner\scanner_fig2.png)

解决方案: 只用一个 Scanner 对象接收标准输入, 不要重复创建.    
## 坑2: 为什么只关闭了一个 Scanner, 其余的 Scanner 都不能用了?
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
![](\content\images\2018\java_scanner\scanner_fig3.png)
![](\content\images\2018\java_scanner\scanner_fig4.png)
## 坑3: 为什么没有关闭 Scanner, 但新的 Scanner 也会报错?
```java
import java.util.Scanner;
public class Main{
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
![](\content\images\2018\java_scanner\console_trap3.png)



![](\content\images\2018\java_scanner\scanner_fig5.png)
![](\content\images\2018\java_scanner\scanner_fig6.png)
![](\content\images\2018\java_scanner\scanner_fig7.png)