---
title: ClassLoader  
date: 2019-02-15 13:56:44  
categories:
- Java
- ClassLoader  

tags:
- Java
- ClassLoader
---
# ClassLoader
> 类加载器   
> file： ClassLoader.txt   
> 参考:[探秘Java9之类加载](https://yq.aliyun.com/articles/518315)  

## 1.8 and earlier version
code: java [1.8]   
## BootstrapClassLoader,ExtClassLoader,AppClassLoader  
```java
static class AppClassLoader extends URLClassLoader
static class ExtClassLoader extends URLClassLoader
```
### ClassLoader
```java
<tt>Class</tt> objects for array classes are not created by class loaders, but are created automatically as required by the Java runtime.    
The class loader for an array class, as returned by {@link Class#getClassLoader()} is the same as the class loader for its element type; if the element type is a primitive type, then the array class has no class loader.    

<p> The <tt>ClassLoader</tt> class uses a delegation model to search for classes and resources.  
Each instance of <tt>ClassLoader</tt> has an associated parent class loader.  
When requested to find a class or resource, a <tt>ClassLoader</tt> instance will delegate the search for the class or resource to its parent class loader
before attempting to find the class or resource itself.  
The virtual machine's built-in class loader, called the "bootstrap class loader", does not itself have a parent but may serve as the parent of a <tt>ClassLoader</tt> instance.
```
- Bootstrap ClassLoader 最顶层的加载类，主要加载核心类库，%JRE_HOME%\lib下的rt.jar、resources.jar、charsets.jar和class等。另外需要注意的是可以通过启动jvm时指定-Xbootclasspath和路径来改变Bootstrap ClassLoader的加载目录。比如java -Xbootclasspath/a:path被指定的文件追加到默认的bootstrap路径中。我们可以打开我的电脑，在上面的目录下查看，看看这些jar包是不是存在于这个目录。   

- Extention ClassLoader 扩展的类加载器，加载目录%JRE_HOME%\lib\ext目录下的jar包和class文件。还可以加载-D java.ext.dirs选项指定的目录。   

- AppClass Loader ` ClassLoader.getSystemClassLoader()` 也称为SystemAppClass 加载当前应用的classpath的所有类。

***ExtClassLoader与AppClassLoader代码见：lib\rt.jar!\sun\misc\Launcher.class***   

<!-- more -->

### loadClass
加载类--->双亲委派机制体现（查看是否已经加载，父加载器是否加载   顶级加载器加载，。。。，查找）
```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException{
    synchronized (getClassLoadingLock(name)) {//lock .....
        // First, check if the class has already been loaded
        // 是否已经加载
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    //父加载器加载类
                    c = parent.loadClass(name, false);
                } else {
                    //BootstrapClass loader 加载过（BootstrapClassLoader JVM一部分，C/C++代码实现）
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }
            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);//父类无法加载，自行查找
                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            //links the specified class
            resolveClass(c);
        }
        return c;
    }
}
```
## 使用
自定义加载路径（本地文件，网络资源）   
```java
File file=new File("f:/code/mine");
System.out.println(file.isDirectory()+" "+file.toURI());
new URLClassLoader(new URL[]{new URL("file:/f:/code/mine/")}, ClassLoader.getSystemClassLoader())  
```
### 自定义ClassLoader
```java
public class MClassLoader extends ClassLoader {
    private String mLibPath;

    public MClassLoader(String mLibPath) {
        this.mLibPath = mLibPath;
        System.out.println(getParent());
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            byte[] b = loadClassData(name);
            if (b != null) {
                return defineClass(name, b, 0, b.length);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return super.findClass(name);
    }

    private byte[] loadClassData(String name) throws IOException {
        String fileName = getFileName(name);
        File resFile = new File(mLibPath, fileName);
        if (resFile != null) {
            System.out.println("loading ... "+resFile.getAbsolutePath());
            return Files.readAllBytes(Paths.get(resFile.getAbsolutePath()));
        }
        return null;
    }

    private String getFileName(String name) {
        int pos = name.lastIndexOf('.');
        if (pos == -1)
            return name + ".class";
        else
            return name.substring(pos + 1) + ".class";
    }
}
```
### Test
```java
public class LoaderTest {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, IllegalAccessException, InstantiationException, InvocationTargetException {
        MClassLoader loader=new MClassLoader("F:/code/mine");
        Class cl=loader.loadClass("com.yue.test.Hello");
        Method say=cl.getDeclaredMethod("say");
        say.invoke(cl.newInstance());
    }
}
```
![](load_Hello.jpg)   
### Result
```
sun.misc.Launcher$AppClassLoader@18b4aac2
loading ... F:\code\mine\Hello.class
Hello Test ClassLoader !

Process finished with exit code 0
```


## Question
重名包/类 冲突处理！！！----> 重写load方法。。。but defineClass
### 尝试1
```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            // If still not found, then invoke findClass in order
            // to find the class.
            c = findClass(name);
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
@Override
protected Class<?> findClass(String name) throws ClassNotFoundException {
   try {
       byte[] b = loadClassData(name);
       if (b != null) {
           return defineClass(name, b, 0, b.length);//.....
       }
   } catch (IOException e) {
       e.printStackTrace();
   }
   return super.findClass(name);
}
```
在native method:defineClass时会查找父类是否加载，如果没有，则进行加载。<!-- 该过程存在于native方法defineClass -->       
when defineClass --> 自定义ClassLoader下--> 在制定目录下查找父类 如Object.class时，由于制定目录下无此class   因此 Exception    

### 尝试2
```java
protected Class<?> getClass(String name) throws ClassNotFoundException {
    try {
        byte[] b = loadClassData(name);

        ClassLoader cl = getParent();//AppClassLoader
        Method defineClassMethod = ClassLoader.class.getDeclaredMethod("defineClass", byte[].class, int.class, int.class);
        defineClassMethod.setAccessible(true);
        return (Class<?>) defineClassMethod.invoke(cl,b,0,b.length);
    } catch (IOException | NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {
        e.printStackTrace();
    }
    return null;
}
```
先加载待加载类字节码byte然后利用父加载器进行定义

### 改进
- 重复加载
- 多线程


## Java
ClassLoader: ContextClassLoader   
Thread .setContextClassLoader/.getContextClassLoader

## Java 9 and later
Java9带来了模块化系统，同时类加载机制也进行了调整，Java9中的类加载器，变化仅仅是ExtClassLoader消失了且多了PlatformClassLoader，JVM规范里5.3 Creation and Loading部分详细描述了类加载，这里简单说下，规范里把类加载器分为两类，一类是由虚拟机提供的启动类加载器，另一类是由用户自定义的类加载器，注意数组的创建不是类加载器创建的，而是由虚拟机直接创建的。而加载又分为两种情况：defining loader和initiating loader，defining loader只加载不初始化，initiating loader是加载并初始化。在运行时一个类或接口是否唯一不是取决于其二进制名称，而是二进制名称和defining其的类加载器的组合，这些和之前保持一致的，那具体区别在哪？其中JVM规范5.3.6 Modules and Layers有详细说明，增加了Layer（层）的概念，用Layer表示模块集，其实Layer和类加载器是对应的，将启动类加载器加载的模块归一Layer，用户自定义类加载器加载的模块归到另一Layer，Layer也有委托的概念。

![类加载器](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/88eef4211959d2e72546d7bedc361e40.png "类加载器")   

....
