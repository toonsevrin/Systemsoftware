# A9: Bounded buffer

If a thread tries to insert itself while the buffer is full, the `while ( spaces == 0 ) { Thread.sleep( 10 ) }` makes the thread "queue" and poll ~every 10ms. Same goes for removing while the buffer is empty (`while ( items == 0 ){ ... }`).

Problem with this solution: The functions are not atomic/syncronized. This can lead to concurrentmodificationexceptions and race conditions. Solutions: Semaphores/synchronized keyword/AtomicBoolean/...

Replicate the problem: Remove the Thread.sleep(10) line in the while loops and the program will go in a deadlock after a while.



```java

    private static class BoundedBuffer {
        static final int BUFFER_SIZE = 5;             // constant holding size of the buffer

        private Object[] buffer;                      // buffer as an Object array

        private int spaces;
        private int items;
        private int in;
        private int out;

        private BoundedBuffer() {
            buffer = new Object[BUFFER_SIZE];
            spaces = BUFFER_SIZE;
            items = 0;
            in = 0;
            out = 0;
        }

        public void insert(Object item) throws InterruptedException {
            while (spaces == 0) {
                Thread.sleep(10);
            }

            buffer[in] = item;
            System.out.println("item " + item + " inserted in slot " + in);
            in = (in + 1) % BUFFER_SIZE;
            items = items + 1;
            spaces = spaces - 1;
        }

        public Object remove() throws InterruptedException {
            Object item;

            while (items == 0) {
                Thread.sleep(10);
            }

            item = buffer[out];
            System.out.println("item " + item + " removed from slot " + out);
            out = (out + 1) % BUFFER_SIZE;
            items = items - 1;
            spaces = spaces + 1;

            return item;
        }
    }
```

