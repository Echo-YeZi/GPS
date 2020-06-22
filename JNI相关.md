# JNI

### 1. 参考资料

> [Android JNI(一)——NDK与JNI基础](https://www.jianshu.com/p/87ce6f565d37)



### 2. 简介

> Java Native Interface，即Java本地接口，它是Java平台的一个特性(并不是Android系统特有的)。



### 3. 用途

> 其实主要是定义了一些JNI函数，让开发者可以通过调用这些函数实现Java代码调用C/C++的代码，C/C++的代码也可以调用Java的代码



### 4. 优势

> JNI确保了二进制兼容性，应用程序兼容性和Java虚拟机兼容性（不仅限于Davik）,进而JNI处理后的c/c++的代码可以在大部分平台上执行

> 二进制兼容性是一种程序兼容性类型，允许一个程序在不改变其可执行文件的条件下在不同的编译环境中工作



### 5. 角色

1. C/C++代码，通常为dll和so
2. JNI，即本地方法接口
3. Java代码，即具体业务类

![img](https://upload-images.jianshu.io/upload_images/5713484-62edbafafd3f3b2d.png?imageMogr2/auto-orient/strip|imageView2/2/w/788/format/webp)

![img](https://upload-images.jianshu.io/upload_images/5713484-87fe79f0796a77a3.png?imageMogr2/auto-orient/strip|imageView2/2/w/891/format/webp)



### 6. 命名规则

> 如：
>
> ```undefined
> JNIExport jstring JNICALL Java_com_example_hellojni_MainActivity_stringFromJNI( JNIEnv* env,jobject thiz ) 
> ```
>
> **`jstring`** 是**返回值类型**
>  **`Java_com_example_hellojni`** 是**包名**
>  **`MainActivity`** 是**类名**
>  **`stringFromJNI`** 是**方法名**
>
> 其中**`JNIExport`**和**`JNICALL`**是不固定保留的关键字不要修改



### 7. 使用

> 参考: https://www.runoob.com/w3cnote/jni-getting-started-tutorials.html

> 一般先将写好的C/C++代码编译成对应平台的动态库，再进行调用
>
> - windows一般是dll文件
> - linux一般是so文件等，Android属于linux分支

> 1. 在Java中先声明一个native方法
> 2. 编译Java源文件javac得到.class文件
> 3. 通过javah -jni命令导出JNI的.h头文件
> 4. 使用Java需要交互的本地代码，实现在Java中声明的Native方法（如果Java需要与C++交互，那么就用C++实现Java的Native方法。）
> 5. 将本地代码编译成动态库(Windows系统下是.dll文件，如果是Linux系统下是.so文件，如果是Mac系统下是.jnilib)
> 6. 通过Java命令执行Java程序，最终实现Java调用本地代码。
>
> > javah 是JDK自带的一个命令，-jni参数表示将class 中用到native 声明的函数生成JNI 规则的函数

1. 编写Java测试类

   ```java
   public class JNIDemo {
       
       //定义一个方法，该方法在C中实现
       public native void testHello();
       
       public static void main(String[] args){
           //加载C文件
           System.loadLibrary("TestJNI");
           JNIDemo jniDemo = new JNIDemo();
           jniDemo.testHello();
       }
   
   }
   ```

2. 利用java类生成C头文件

   > 在java测试类的工程的bin目录下执行
   >
   > `javah -classpath . -jni com.aijiao.test.JNIDemo`

3. 编写C代码

   > 1. 创建C/C++项目,应用程序类型为DLL
   >
   > 2. 复制下面文件到C工程目录
   >
   >    - JDK目录的include目录下有一个jni.h
   >    - include的win32目录下有个jni_md.h
   >    - java工程的bin目录下的C头文件
   >
   > 3. 对C项目添加头文件-现有项，选择第二步复制的三个文件
   >
   > 4. 编辑C头文件，将`#include <jni.h>`修改为`#include "jni.h"`
   >
   >    > - #include< > 引用的是编译器的类库路径里面的头文件
   >    >
   >    > - \#include" " 引用的是程序目录的相对路径中的头文件
   >
   > 5. 编辑C项目源文件
   >
   >    ```c++
   >    #include "com_aijiao_test_JNIDemo.h"
   >    #include <iostream>
   >    #include <stdio.h>
   >    
   >    JNIEXPORT void JNICALL Java_com_aijiao_test_JNIDemo_testHello
   >    (JNIEnv *, jobject) {
   >        printf("this is C++ print");
   >    }
   >    ```
   >
   >    > - `Java_com_aijiao_test_JNIDemo_testHello`为Java测试类里定义的方法
   >    > - `com_aijiao_test_JNIDemo.h`为C头文件
   >
   > 6. 配置C工程，解决方案-属性
   >
   >    ![img](https://www.runoob.com/wp-content/uploads/2017/07/1499482021-2464-161623-SocN-2730791.png)

4. 生成项目，根据输出的相关信息找到DLL文件

   ![img](https://www.runoob.com/wp-content/uploads/2017/07/1499482021-6260-161900-Caku-2730791.png)

5. 在Java项目下配置`Java Build Path`-`Library`-`JRE System Library`-`Native Library Location`，填写DLL文件目录路径

6. 运行Java项目

### 8. JNI结构

![img](https://upload-images.jianshu.io/upload_images/5713484-8b84c6a37e1c8967.png?imageMogr2/auto-orient/strip|imageView2/2/w/529/format/webp)

![img](https://upload-images.jianshu.io/upload_images/5713484-48acac24bc7f78a1.png?imageMogr2/auto-orient/strip|imageView2/2/w/834/format/webp)

**示例**

```c++
jdouble Java_pkg_Cls_f__ILjava_lang_String_2 (JNIEnv *env, jobject obj, jint i, jstring s)
{
     const char *str = (*env)->GetStringUTFChars(env, s, 0); 
     (*env)->ReleaseStringUTFChars(env, s, str); 
     return 10;
}
```

> - *env：一个接口指针
> - obj：在本地方法中声明的对象引用
> - i和s：用于传递的参数



### 9. JNI原理

1. **JavaVM**

   > JavaVM是Java虚拟机在JNI层的代表，JNI全局仅仅有一个JavaVM结构中封装了一些函数指针（或叫函数表结构），JavaVM中封装的这些函数指针主要是对JVM操作接口.
   >
   > >  C++中有对JNIInvokeInterface_进行了一次封装，比C中少了一个参数，因此JNI代码更推荐使用C++来编写

2. ##### JNIEnv

   > JNIEnv是当前Java线程的执行环境，
   >
   > 一个JVM对应一个JavaVM结构，而一个JVM中可能创建多个Java线程，每个线程对应一个JNIEnv结构，它们保存在线程本地存储TLS中。
   >
   > 不同的线程的JNIEnv是不同，也不能相互共享使用。
   >
   > JNIEnv结构也是一个函数表，在本地代码中通过JNIEnv的函数表来操作Java数据或者调用Java方法。
   >
   > 只要在本地代码中拿到了JNIEnv结构，就可以在本地代码中调用Java代码。



### 10 引用

1. **局部引用(Local Reference)**

   > 局部引用，也成本地引用，通常是在函数中创建并使用。会阻止GC回收所有引用对象。

2. **全局引用(Global Reference)**

   > 全局引用可以跨方法、跨线程使用，直到被开发者显式释放。类似局部引用，一个全局引用在被释放前保证引用对象不被GC回收。和局部应用不同的是，没有俺么多函数能够创建全局引用。能创建全部引用的函数只有NewGlobalRef，而释放它需要使用ReleaseGlobalRef函数

3. **弱全局引用(Weak Global Reference)**

   > 是JDK 1.2 新增加的功能，与全局引用类似，创建跟删除都需要由编程人员来进行，这种引用与全局引用一样可以在多个本地带阿妈有效，不一样的是，弱引用将不会阻止垃圾回收期回收这个引用所指向的对象，所以在使用时需要多加小心，它所引用的对象可能是不存在的或者已经被回收。

4. ##### 引用比较

   > 在给定两个引用，不管是什么引用，我们只需要调用IsSameObject函数来判断他们是否是指向相同的对象
   >
   > `(*env)->IsSameObject(env, obj1, obj2)` 是相同对象则返回**JNI_TRUE(或者1)**，否则返回**JNI_FALSE(或者0)**

   