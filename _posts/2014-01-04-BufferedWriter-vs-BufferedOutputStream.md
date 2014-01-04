---
layout: post
title: BufferedWriter和BufferedOutputStream性能比较
category: program
---

~~~~
package com.io;
import java.io.BufferedOutputStream;
import java.io.BufferedWriter;
import java.io.File;
import java.io.FileOutputStream;
import java.io.FileWriter;
public class TestFileWriter {
    
    private static int COUNT = 100000000;
    
    private static String FILE_NAME = "/var/testFile.txt";
    public static void main(String[] args) throws Exception {
        execute(new OutputStreamExecutor(FILE_NAME, COUNT));
        
        execute(new WriterExecutor(FILE_NAME, COUNT));
    }

    public static void execute(Executor executor) throws Exception {
        
        String text = "hello world";
        long startTime = System.currentTimeMillis();
        executor.execute(text);
        long endTime = System.currentTimeMillis();
        
        System.out.println("" + executor.getClass().getSimpleName() + "耗时：" + (endTime - startTime));
    }

    static interface Executor {
        public void execute(String text) throws Exception;
    }

    static class OutputStreamExecutor implements Executor {
        private String fileName;
        private int count;
        OutputStreamExecutor(String fileName, int count) {
            this.fileName = fileName;
            this.count = count;
        }
        @Override
        public void execute(String text) throws Exception {
            File file = new File(fileName);
            BufferedOutputStream bos = new BufferedOutputStream(
                    new FileOutputStream(file));
            for (int i = 0; i < count; i++) {
                bos.write(text.getBytes());
            }
            
            bos.flush();
            bos.close();
        }
    }
    static class WriterExecutor implements Executor {
        private String fileName;
        private int count;
        
        WriterExecutor (String fileName, int count) {
            this.fileName = fileName;
            this.count = count;
        }

        @Override
        public void execute(String text) throws Exception {
            
            File file = new File(fileName);
            
            BufferedWriter writer = new BufferedWriter(new FileWriter(file));
            for (int i = 0; i < count; i++) {
                writer.write(text);
            }
            
            writer.flush();
            writer.close();
        }
    }
}  
~~~~
 
测试结果如下：  

~~~~
OutputStreamExecutor耗时：313
WriterExecutor耗时：122
~~~~

差别还是比较大的，BufferedWriter表现比BufferedOutputStream优秀，查看相关的源码，BufferedWriter最后调用的是StreamEncoder写数据，而它借用了nio中的CharBuffer：
 
~~~~
    public void write(char cbuf[], int off, int len) throws IOException {
   synchronized (lock) {
       ensureOpen();
            if ((off < 0) || (off > cbuf.length) || (len < 0) ||
                ((off + len) > cbuf.length) || ((off + len) < 0)) {
                throw new IndexOutOfBoundsException();
            } else if (len == 0) {
                return;
            }
 
       if (len >= nChars) {
      /* If the request length exceeds the size of the output buffer,
         flush the buffer and then write the data directly.  In this
         way buffered streams will cascade harmlessly. */
      flushBuffer();
      out.write(cbuf, off, len);
      return;
       }
 
       int b = off, t = off + len;
       while (b < t) {
      int d = min(nChars - nextChar, t - b);
      System.arraycopy(cbuf, b, cb, nextChar, d);
      b += d;
      nextChar += d;
      if (nextChar >= nChars)
          flushBuffer();
       }
   }
}

~~~~
 
注意：不管是BufferedWriter和BufferedOutputStream都需要在最后调用flush，或者close。
