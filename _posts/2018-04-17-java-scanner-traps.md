---
layout: post
title: Java 中 Scanner 使用的一些坑
key: 2018-04-17-java-scanner-traps

---       
在 Java 中, 我们经常使用 ```Scanner``` 类来获取用户输入.

<!--more-->
## 坑1: 为什么我的输入只有行内开始的一个数, 后面的数都不见了？
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
这通常是由于重复创建了多个 Scanner 导致的. Scanner 被创建时, 会 
![](\content\images\2018\java_scanner\scanner_fig1.png)
![](\content\images\2018\java_scanner\scanner_fig2.png)
## 坑2: 为什么只关闭了一个 Scanner, 其余的 Scanner 都不能用了?
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

```text
This is test file line 1
This is test file line 2
```



![](\content\images\2018\java_scanner\scanner_fig5.png)
![](\content\images\2018\java_scanner\scanner_fig6.png)
![](\content\images\2018\java_scanner\scanner_fig7.png)