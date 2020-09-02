---
title: MappedByteBuffer
date: 2020-09-02 12:20:09
categories:
- Java
- NIO
tags:
- Java
- NIO
---
### 目录

*   [New I/O](#New_IO_2)

*   [ByteBuffer](#ByteBuffer_33)

*   [Direct Buffers](#Direct_Buffers_64)
*   [MappedByteBuffer](#MappedByteBuffer_80)

*   [示例](#_148)
*   [参考理解](#_223)

New I/O
-------

> 旧的 I/O 包已经使用 nio 重新实现过，以便充分利用这种速度提高

速度的提高来自于所使用的数据集结构更接近于操作系统执行的 I/O 方式：**通道**和**缓冲器**，唯一直接与通道交互的缓冲器`ByteBuffer`。旧的 I/O 库中的`FileInputStream`,`FileOutputStream`,`RandomAccessFile`可以用来生成`FileChannel`;`Reader`和`Writer`不能用于产生通道，但是在`java.nio.channels.Channels`类中提供了在通道中产生`Reader`和`Writer`的方法。

![](https://img-blog.csdnimg.cn/20200901203630879.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0NjIwODU1,size_16,color_FFFFFF,t_70#pic_center)

```java
/**
 * Constructs a reader that decodes bytes from the given channel using the given decoder.
 */
public static Reader newReader(ReadableByteChannel ch, CharsetDecoder dec, int minBufferCap){
     checkNotNull(ch, "ch");
     return StreamDecoder.forDecoder(ch, dec.reset(), minBufferCap);// new StreamDecoder(...);
}
public static Reader newReader(ReadableByteChannel ch, String csName){
    checkNotNull(csName, "csName");
    return newReader(ch, Charset.forName(csName).newDecoder(), -1);
}

public static Writer newWriter(final WritableByteChannel ch, final CharsetEncoder enc, final int minBufferCap){
    checkNotNull(ch, "ch");
    return StreamEncoder.forEncoder(ch, enc.reset(), minBufferCap);// ..
}
public static Writer newWriter(WritableByteChannel ch, String csName){
     checkNotNull(csName, "csName");
     return newWriter(ch, Charset.forName(csName).newEncoder(), -1);
 }
```

### ByteBuffer

A byte buffer is either _direct_ or _non-direct_.

*   Given a direct byte buffer, the Java virtual machine will make a best effort to perform native I/O operations directly upon it. That is, it will attempt to avoid copying the buffer’s content to (or from) an intermediate buffer before (or after) each invocation of one of the underlying operating system’s native I/O operations.

*   The contents of direct buffers may reside outside of the normal garbage-collected heap, and so their impact upon the memory footprint of an application might not be obvious.

*   It is therefore recommended that _**direct buffers be allocated primarily for large, long-lived buffers that are subject to the underlying system’s native I/O operations.**_

*   In general it is best to allocate direct buffers **only when they yield a measureable gain in program performance.**


```java
ByteBuffer(int mark, int pos, int lim, int cap,   // package-private
                 byte\[\] hb, int offset){
		 	super(mark, pos, lim, cap);
			this.hb = hb;     		///   Non-null only for heap buffers
			this.offset = offset;	///   
}
Buffer(int mark, int pos, int lim, int cap) {       // package-private
		    if (cap < 0)
		          throw new IllegalArgumentException("Negative capacity: " + cap);
	        this.capacity = cap;
	        limit(lim);
	        position(pos);
	        if (mark >= 0) {
	           if (mark > pos)
	              throw new IllegalArgumentException("mark > position: (" + mark + " > " + pos + ")");
	           this.mark = mark;
	       }
  }
```

#### Direct Buffers

```
// Allocates a new direct byte buffer.
// this method typically have somewhat higher allocation and deallocation costs than non-direct buffers. 更高的分配和释放成本
public static ByteBuffer allocateDirect(int capacity) {
 	return new DirectByteBuffer(capacity);
}

// Maps a region of this channel's file directly into memory.
//Read/write: Changes made to the resulting buffer will eventually be propagated to the file; they may or may not be made visible to other programs that have mapped the same file.
//它们可能对映射了同一文件的其他程序可见，也可能不可见。
//Private: Changes made to the resulting buffer will not be propagated to the file and will not be visible to other programs that have mapped the same file; instead, they will cause private copies of the modified portions of the buffer to be created.                                  
public abstract MappedByteBuffer map(MapMode mode, long position, long size) throws IOException;
```

#### MappedByteBuffer

> A direct byte buffer whose content is a memory-mapped region of a file.  
> _A mapped byte buffer and the file mapping that it represents remain valid until the buffer itself is garbage-collected._

For most operating systems, _**mapping a file into memory**_ is _more expensive_ than reading or writing **a few tens of kilobytes** of data via the usual {@link #read read} and {@link #write write} methods. From the standpoint of performance it is generally _**only worth mapping relatively large files into memory.**_

```java
// Tells whether or not this buffer's content is resident in physical memory.
// A return value of <tt>false</tt> does not necessarily imply that the buffer's content is not resident in physical memory.
// The returned value is a hint, rather than a guarantee,
// because the underlying operating system may
// have paged out some of the buffer's data by the time that an invocation of this method returns.
public final boolean isLoaded() {
		checkMapped();
        if ((address == 0) || (capacity() == 0))
            return true;
        long offset = mappingOffset();
        long length = mappingLength(offset);
        return isLoaded0(mappingAddress(offset), length, Bits.pageCount(length));
}
private native boolean isLoaded0(long address, long length, int pageCount);

// Loads this buffer's content into physical memory.
// This method makes a best effort to ensure that ...
public final MappedByteBuffer load() {
        checkMapped();
        if ((address == 0) || (capacity() == 0))
            return this;
        long offset = mappingOffset();
        long length = mappingLength(offset);
        load0(mappingAddress(offset), length);

        //  A checksum is computed as we go along to prevent the compiler from otherwise
        // considering the loop as dead code.
        Unsafe unsafe = Unsafe.getUnsafe();
        int ps = Bits.pageSize();
        int count = Bits.pageCount(length);
        long a = mappingAddress(offset);
        byte x = 0;
        for (int i=0; i<count; i++) {
            x ^= unsafe.getByte(a);//Read a byte from each page to bring it into memory.
            a += ps;
        }
        if (unused != 0)
            unused = x;

        return this;
}
private native void load0(long address, long length);


// Forces any changes made to this buffer's content to be written to the storage device containing the mapped file.
// If the file mapped into this buffer resides on a local storage device
// then when this method returns it is guaranteed that
// all changes made to the buffer since it was created, or since this method was last invoked, will have been written to that device.
//
// If the file does not reside on a local device then no such guarantee is made.
public final MappedByteBuffer force() {// read/write mode
        checkMapped();
        if ((address != 0) && (capacity() != 0)) {
            long offset = mappingOffset();
            force0(fd, mappingAddress(offset), mappingLength(offset));
        }
        return this;
}
```

示例
--

> 将 E://test 目录下的大文件进行读写

```java
//Main.java
private static void testFiles() {
        ExecutorService executorService = Executors.newFixedThreadPool(12);
       	//....
                                       FileChannel fileChannel = new RandomAccessFile(file1, "rw").getChannel();
                                for (int j = 0; j < 8; j++) {
                                    MappedByteBuffer buffer = fileChannel.map(FileChannel.MapMode.READ\_WRITE, j \* block, block);
                                    executorService.execute(new RWTest(file1.getName(), fileChannel, buffer, j \* block, block));
                                }
                            }  // 文件太小
                   }
//RWTest.java
public void run() {
        long start = System.currentTimeMillis();
        FileLock fileLock = null;
        while (true) {
            try {
                fileLock = fileChannel.tryLock(position, block, false);
                break;
            } catch (IOException e) {
                System.out.println("there is another thread here.");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e1) {
                    e1.printStackTrace();
                }
            }
        }

        // do something ...
        int index = 0;
        while (buffer.hasRemaining()) {
            buffer.put((byte) (buffer.get(index) ^ chaos\[index % 16\]));//数据读写
            index++;
        }
        try {
            if (fileLock != null)
                fileLock.release();
        } catch (IOException e) {
            e.printStackTrace();
        }

        long stop = System.currentTimeMillis();
        System.out.println(String.format("%s 耗费 %d毫秒", this, stop - start));
}
```

![](https://img-blog.csdnimg.cn/20200901213234949.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0NjIwODU1,size_16,color_FFFFFF,t_70#pic_center)  
![](https://img-blog.csdnimg.cn/20200901213615877.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0NjIwODU1,size_16,color_FFFFFF,t_70#pic_center)  
![](https://img-blog.csdnimg.cn/20200901213741243.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0NjIwODU1,size_16,color_FFFFFF,t_70#pic_center)

```
// 首次运行
Thread pool-1-thread-3-14 	position:643952840  block: 321976420    covert:aa.mkv  耗费 25074毫秒
Thread pool-1-thread-7-18 	position:1931858520  block: 321976420    covert:aa.mkv  耗费 25310毫秒
Thread pool-1-thread-8-19 	position:2253834940  block: 321976420    covert:aa.mkv  耗费 26502毫秒
Thread pool-1-thread-2-13 	position:321976420  block: 321976420    covert:aa.mkv  耗费 27924毫秒
Thread pool-1-thread-4-15 	position:965929260  block: 321976420    covert:aa.mkv  耗费 28847毫秒
Thread pool-1-thread-6-17 	position:1609882100  block: 321976420    covert:aa.mkv  耗费 28859毫秒
Thread pool-1-thread-5-16 	position:1287905680  block: 321976420    covert:aa.mkv  耗费 29086毫秒
Thread pool-1-thread-1-12 	position:0  block: 321976420    covert:aa.mkv  耗费 29309毫秒

// 第二次转码，在内存中，速度极快
Thread pool-1-thread-2-13 	position:321976420  block: 321976420    covert:aa.mkv  耗费 1248毫秒
Thread pool-1-thread-7-18 	position:1931858520  block: 321976420    covert:aa.mkv  耗费 1275毫秒
Thread pool-1-thread-4-15 	position:965929260  block: 321976420    covert:aa.mkv  耗费 1285毫秒
Thread pool-1-thread-6-17 	position:1609882100  block: 321976420    covert:aa.mkv  耗费 1290毫秒
Thread pool-1-thread-5-16 	position:1287905680  block: 321976420    covert:aa.mkv  耗费 1297毫秒
Thread pool-1-thread-8-19 	position:2253834940  block: 321976420    covert:aa.mkv  耗费 1300毫秒
Thread pool-1-thread-3-14 	position:643952840  block: 321976420    covert:aa.mkv  耗费 1311毫秒
Thread pool-1-thread-1-12 	position:0  block: 321976420    covert:aa.mkv  耗费 1348毫秒
```

参考理解
----

[占小狼 - 深入浅出 MappedByteBuffer](https://www.jianshu.com/p/f90866dcbffc)  
[mrguozp-MappedByteBuffer](https://www.cnblogs.com/guozp/p/10470431.html)

传统的基于文件流的方式读取文件方式是系统指令调用，文件数据首先会被读取到进程的内核空间的缓冲区，而后复制到进程的用户空间，这个过程中存在两次数据拷贝；而内存映射方式读取文件的方式，也是系统指令调用，在产生缺页中断后，CPU 直接从磁盘文件 load 数据到进程的用户空间，只有一次数据拷贝。

*   性能分析  
    从代码层面上看，从硬盘上将文件读入内存，都要经过文件系统进行数据拷贝，并且数据拷贝操作是由文件系统和硬件驱动实现的，理论上来说，拷贝数据的效率是一样的。  
    但是通过内存映射的方法访问硬盘上的文件，效率要比 read 和 write 系统调用高，这是为什么？  
    read() 是系统调用，首先将文件从硬盘拷贝到内核空间的一个缓冲区，再将这些数据拷贝到用户空间，实际上进行了两次数据拷贝；  
    map() 也是系统调用，但没有进行数据拷贝，当缺页中断发生时，直接将文件从硬盘拷贝到用户空间，只进行了一次数据拷贝。  
    所以，采用内存映射的读写效率要比传统的 read/write 性能高。

*   总结  
    MappedByteBuffer 使用虚拟内存，因此分配 (map) 的内存大小不受 JVM 的 - Xmx 参数限制，但是也是有大小限制的。  
    如果当文件超出 1.5G 限制时，可以通过 position 参数重新 map 文件后面的内容。  
    MappedByteBuffer 在处理大文件时的确性能很高，但也存在一些问题，如内存占用、文件关闭不确定，被其打开的文件只有在垃圾回收的才会被关闭，而且这个时间点是不确定的。

*   使用注意  
    关于 MappedByteBuffer 资源释放问题  
    [langgufu314-java 大文件读写操作，java nio 之 MappedByteBuffer，高效文件 / 内存映射](https://blog.csdn.net/langgufu314/article/details/84627609)


```java
//文件复制,只作为示例  此类文件复制:transferFrom transferTo
new Tester("Questions here") {
      	@Override
        protected void test() throws IOException {
            File source = new File("E://test//aa.mkv");
            File dest = new File("E://aa.mkv");
            FileChannel in = null, out = null;
            try {
                in = new FileInputStream(source).getChannel();
                out = new FileOutputStream(dest).getChannel();
                long size = in.size() > 2147483647L ? 2147483647 : in.size();
                MappedByteBuffer buf = in.map(FileChannel.MapMode.READ\_ONLY, 0, size);
                out.write(buf);
                in.close();
                out.close();
                System.out.println(source.delete());//文件复制完成后，删除源文件
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                in.close();
                out.close();
            }
       }
}
```

运行结果：

```
Questions here : false  //无法成功删除
耗时20050ms
```

主要原因是变量 buf 仍然有源文件的句柄，文件处于不可删除状态。既然 MappedByteBuffer 是从 FileChannel 中 map()出来的，为 什么它又不提供 unmap()呢？SUN 自己也没有讲清楚为什么。O’Reilly 的 <<Java NIO>> 中说是因为 "安全" 的原因，但是到底 unmap()会怎么不安全，作者也没有讲清楚。  
在 sun 网站也有相应的 BUG 报告：bug id:4724038 [链接](http://bugs.sun.com/bugdatabase/view_bug.do?bug_id=4724038)，但是 sun 自己不认为是 BUG，而只是一个 RFE(Request For Enhancement)，有待改进。

解决方案：

> 利用反射获取 cleaner 方法进行清理

```java
public static void clean(final Object buffer) throws Exception {
        AccessController.doPrivileged(new PrivilegedAction() {
            public Object run() {
                try {
                    Method getCleanerMethod = buffer.getClass().getMethod("cleaner", new Class\[0\]);//获取 cleaner 方法
                    getCleanerMethod.setAccessible(true);
                    sun.misc.Cleaner cleaner = (sun.misc.Cleaner) getCleanerMethod.invoke(buffer, new Object\[0\]);//ok...
                    cleaner.clean();
                } catch (Exception e) {
                    e.printStackTrace();
                }
                return null;
            }
        });
    }
```
