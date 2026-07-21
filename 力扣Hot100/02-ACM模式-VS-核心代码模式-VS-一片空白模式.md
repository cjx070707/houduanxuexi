# ACM 模式 VS 核心代码模式 VS 一片空白模式

写代码时准确的说有三种模式/情况

1、核心代码模式，即leetcode刷题时，只需要写出核心函数的功能即可

2、ACM模式，已给定main函数，需要使用 Scanner 读取输入 和 System.out 输出

3、一片空白模式，啥都没有，全部都需要自己写

这几种模式会在什么情况下遇到呢？

`核心代码模式`和`一片空白模式` 会在 面试手撕算法时遇到

`ACM模式` 只会在 笔试算法时遇到

### ACM 模式

网上经常有人问：去哪里练习ACM模式 ？

我是感觉大家存在一些误解，因为ACM模式不难，并且它不是最重要的，ACM 模式只是在笔试时会遇到。

按重要程度来说：一片空白模式 >> 核心代码模式 >> ACM 模式

**ACM 模式 只需要注意 如何使用 Scanner 读取正确的输入即可，并不需要你平时就用ACM模式去刷题练习**

```java
import java.util.Scanner;

public class Main {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        
        // 读取整数
        int n = sc.nextInt();
        
        // 读取数组
        int[] arr = new int[n];
        for (int i = 0; i < n; i++) {
            arr[i] = sc.nextInt();
        }
        
        // 调用解决方案
        int result = solution(arr);
        
        // 输出结果
        System.out.println(result);
    }
    
    public static int solution(int[] nums) {
        // 解决方案
        return 0;
    }
}
```

### 核心代码模式

这个是最简单的模式，即leetcode刷题时，只需要写出核心函数的功能即可

**在面试手撕算法时，30%的概率是这种模式**

```java
// LeetCode风格，只写解决方案
class Solution {
    public int functionName(int[] nums) {
        // 只需要实现这个函数
        return 0;
    }
}
```

### 一片空白模式

这个是最重要的，需要特别注意，在面试手撕算法时，70%的概率是这种模式

一片空白模式，啥都没有，就相当于你打开了一个 txt 文本文档

全部都需要自己写，需要自己写main函数，写测试用例(假数据)，测试运行输出，需要自己导入包

如果使用到了 链表、二叉树 需要自己去定义 Class ，特别注意：有些同学可能不会写构造函数

#### 测试用例

面试时写算法题，面试官需要看到你的测试用例，这时候就是自己在代码里写死假数据，再输出结果，这样就可以了。不需要像ACM模式那样（即不需要使用Scanner去读取输入）。

#### 导入包

一般情况下在最上面写 import java.util.\*; 即可

如果实在碰到 需要使用的类，不知道在哪个包，可以和面试官说一下，提出请求：去百度查一下类所在的包

#### 自定义类和构造函数

当你使用到链表或二叉树等数据结构时，需要自己去定义 Class ，特别注意：有些同学可能不会写构造函数

```java
import java.util.*;

// 空白开始，所有都要自己写
public class InterviewSolution {
    
    // 如果需要链表或二叉树，先定义类
    
    public static void main(String[] args) {
        // 自己创建测试用例
        int[] testCase1 = {1, 2, 3, 4, 5};
        System.out.println("测试用例1: " + Arrays.toString(testCase1));
        System.out.println("结果: " + solution(testCase1));
        
        // 链表测试
        ListNode head = createLinkedList(new int[]{1, 2, 3, 4, 5});
        printLinkedList(head);
    }
    
    public static int solution(int[] nums) {
        // 解决方案
        return 0;
    }
}
```

##### 链表节点定义

```java
// 单向链表
class ListNode {
    int val;
    ListNode next;
    
    // 构造函数（必须掌握！）
    ListNode() {}
    ListNode(int val) { 
        this.val = val; 
    }
    ListNode(int val, ListNode next) { 
        this.val = val; 
        this.next = next; 
    }
}

// 双向链表
class DoubleListNode {
    int val;
    DoubleListNode prev;
    DoubleListNode next;
    
    DoubleListNode() {}
    DoubleListNode(int val) { 
        this.val = val; 
    }
    DoubleListNode(int val, DoubleListNode prev, DoubleListNode next) { 
        this.val = val;
        this.prev = prev;
        this.next = next;
    }
}
```

##### 二叉树节点定义

```java
class TreeNode {
    int val;
    TreeNode left;
    TreeNode right;
    
    // 构造函数（必须掌握！）
    TreeNode() {}
    TreeNode(int val) { 
        this.val = val; 
    }
    TreeNode(int val, TreeNode left, TreeNode right) {
        this.val = val;
        this.left = left;
        this.right = right;
    }
}
```

### 小结

**总的来说，平时使用核心代码模式刷题练习即可。能够写出核心函数的功能才是最重要的。**

ACM模式只需要注意如何正确的使用Scanner读取输入

一片空白模式，也只是多了一些东西需要自己写，并不难，注意一下上述内容即可。
