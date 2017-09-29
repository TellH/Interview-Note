1. ArrayList在每次增加元素（可能是1个，也可能是一组）时，都要调用该方法来确保足够的容量。当容量不足以容纳当前的元素个数时，就设置新的容量为旧的容量的1.5倍加1，如果设置后的新容量还不够，则直接新容量设置为传入的参数（也就是所需的容量），而后用Arrays.copyof()方法将元素拷贝到新的数组
2. 无参构造方法构造的ArrayList的容量默认为10
3. System.arraycopy()方法。该方法被标记了native，调用了系统的C/C++代码，在JDK中是看不到的，但在openJDK中可以看到其源码。该函数实际上最终调用了C语言的memmove()函数，因此它可以保证同一个数组内元素的正确复制和移动，比一般的复制方法的实现效率要高很多，很适合用来批量处理数组。Java强烈推荐在复制大量数组元素时用该方法，以取得更高的效率。
4. public static <T> T[] copyOf(T[] original, int newLength)

```java
    // 快速删除第index个元素    
    private void fastRemove(int index) {    
        modCount++;    
        int numMoved = size - index - 1;    
        // 从"index+1"开始，用后面的元素替换前面的元素。    
        if (numMoved > 0)    
            System.arraycopy(elementData, index+1, elementData, index,    
                             numMoved);    
        // 将最后一个元素设为null    
        elementData[--size] = null; // Let gc do its work    
    }

    // 确定ArrarList的容量。    
    // 若ArrayList的容量不足以容纳当前的全部元素，设置 新的容量=“(原始容量x3)/2 + 1”    
    public void ensureCapacity(int minCapacity) {    
        // 将“修改统计数”+1，该变量主要是用来实现fail-fast机制的    
        modCount++;    
        int oldCapacity = elementData.length;    
        // 若当前容量不足以容纳当前的元素个数，设置 新的容量=“(原始容量x3)/2 + 1”    
        if (minCapacity > oldCapacity) {    
            Object oldData[] = elementData;    
            int newCapacity = (oldCapacity * 3)/2 + 1;    
            //如果还不够，则直接将minCapacity设置为当前容量  
            if (newCapacity < minCapacity)    
                newCapacity = minCapacity;    
            elementData = Arrays.copyOf(elementData, newCapacity);    
        }    
    }
```



http://blog.csdn.net/ns_code/article/details/35568011