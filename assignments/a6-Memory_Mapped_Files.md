# A6 - Memory Mapped Files

Questions:
* Vergelijk klassieke file I/O en memory-mapped files met behulp van FileIO.java.
  * TODO
* Is de performantie voor een memory-mapped file beter?
  * Vooral bij random access wanneer OS de memory-mapped file ondersteund.



`new DataOutputStream(new BufferedOutputStream(new FileOutputStream(new File("temp1.tmp"))));`

Dit kan worden geïnterpreteerd (van binnen naar buiten) als volgt:
- File: creëer een file object met de opgegeven padnaam,
- FileOutputStream: creëer een outputstroom om naar die file te kunnen schrijven, alsook een file descriptor voor de (impliciet) geopende file,
- BufferedOutputStream: gebruik hierbij een buffer van 512 bytes (default),
- DataOutputStream: schrijf primitieve data types (hier: integers) naar de file (i.p.v. een ongestructureerde byte stroom).

Voor een memory-mapped file wordt gebruik gemaakt van een FileChannel. Het resultaat van de mapping is alsof de file in een buffer aanwezig is, die direct kan benaderd worden om integers toe te voegen (put) of te lezen (get).

Memory-mapped file: "Basically, you can tell the OS that some file is the backing store for a certain portion of the process memory." - ayende.com

"A memory-mapped file is a segment of virtual memory that has been assigned a direct byte-for-byte correlation with some portion of a file or file-like resource. This resource is typically a file that is physically present on disk, but can also be a device, shared memory object, or other resource that the operating system can reference through a file descriptor. Once present, this correlation between the file and the memory space permits applications to treat the mapped portion as if it were primary memory." - Wikipedia


```java

import java.io.*;
import java.nio.*;
import java.nio.channels.*;


public class FileIO 
{
    private static int numOfInts = 1000;

    private abstract static class Tester
    {
        private String name;

        public Tester(String name)
        {
            this.name = name; 
        }

        public long runTest()
        {
            System.out.print(name + ": ");
            try
            {
                long startTime = System.currentTimeMillis();
//                long startTime = System.nanoTime();
                test();
                long endTime = System.currentTimeMillis();
//                long endTime = System.nanoTime();
                return (endTime - startTime);
            }
            catch (IOException e)
            {
                throw new RuntimeException(e);
            }
        }
        public abstract void test() throws IOException;
    }

    private static Tester[] tests =
    { 
        new Tester("Sequential write to file")
        {
            public void test() throws IOException
            {
                DataOutputStream dos = new DataOutputStream(
                                        new BufferedOutputStream(
                                         new FileOutputStream(
                                          new File("temp1.tmp"))));

                for(int i = 0; i < numOfInts; i++)
                {
                    dos.writeInt(i);
                }
                if (dos != null)
                {
                    dos.flush();
                }  
                dos.close();
            }
        }, 
        new Tester("Sequential write to memory-mapped file")
        {
            public void test() throws IOException
            {
                RandomAccessFile raf = new RandomAccessFile(
                                        new File("temp2.tmp"), "rw");
                raf.setLength(numOfInts * 4);
                FileChannel fc = raf.getChannel();
                IntBuffer ib = fc.map(FileChannel.MapMode.READ_WRITE, 0,
                                      fc.size()).asIntBuffer();
                for(int i = 0; i < numOfInts; i++)
                {
                    ib.put(i);
                }
                fc.force(false);
                fc.close();
            }
        }, 
        new Tester("Sequential read from file")
        {
            public void test() throws IOException
            {
                DataInputStream dis = new DataInputStream(
                                       new BufferedInputStream(
                                        new FileInputStream("temp1.tmp")));
                for(int i = 0; i < numOfInts; i++)
                {
                    dis.readInt();
                }
                dis.close();
            }
        }, 
        new Tester("Sequential read from memory-mapped file")
        {
            public void test() throws IOException
            {
                FileChannel fc = new FileInputStream(
                                  new File("temp2.tmp")).getChannel();
                IntBuffer ib = fc.map(FileChannel.MapMode.READ_ONLY, 0, 
                                      fc.size()).asIntBuffer();
                while(ib.hasRemaining())
                {
                    ib.get();
                }
                fc.close();
            }
        }, 
        new Tester("Processing a file randomly")
        {
            public void test() throws IOException
            {
                RandomAccessFile raf = new RandomAccessFile(
                                        new File("temp3.tmp"), "rw");
                raf.seek(numOfInts * 4);
                raf.writeInt(0);
                for (int i = 0; i < numOfInts; i++)
                {
                    int number = (int) (Math.random() * numOfInts);
                    raf.seek(number * 4);
                    if (raf.readInt() == 0) 
                    {
                        raf.seek(number * 4);
                        raf.writeInt(number);
                    }
                }
                raf.close();
            }
        }, 
        new Tester("Processing a memory-mapped file randomly")
        {
            public void test() throws IOException
            {
                RandomAccessFile raf = new RandomAccessFile(
                                        new File("temp4.tmp"), "rw");
                raf.setLength(numOfInts * 4);
                FileChannel fc = raf.getChannel();
                IntBuffer ib = fc.map(FileChannel.MapMode.READ_WRITE, 0,
                                      fc.size()).asIntBuffer();

                for(int i = 1; i < numOfInts; i++)
                {
                    int number = (int) (Math.random() * numOfInts);
                    if (ib.get(number) == 0) 
                    {
                        ib.put(number, number);
                    }
                }
                fc.close();
            }
        }
    };
    public static void main(String[] args)
    {
        File oldCopy = new File("temp1.tmp");
        if(oldCopy.exists()) oldCopy.delete();
        oldCopy = new File("temp2.tmp");
        if(oldCopy.exists()) oldCopy.delete();
        oldCopy = new File("temp3.tmp");
        if(oldCopy.exists()) oldCopy.delete();
        oldCopy = new File("temp4.tmp");
        if(oldCopy.exists()) oldCopy.delete();

        if(args.length > 0)
        {
            numOfInts = Integer.parseInt(args[0]);
        }

        for(int i = 0; i < tests.length; i++)
        {
            System.out.println(tests[i].runTest());
        }
    }
}
```
