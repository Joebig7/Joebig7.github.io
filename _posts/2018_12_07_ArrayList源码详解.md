---
title: "ArrayList源码详解"
subtitle: "ArrayList"
layout: post
author: "JoeBig7"
header-style: text
tags:
  - Java
  - Collection
---

## 定义: 一个实现了List接口的可调整大小的数组,实现了List接口所有的可选操作并且可以容纳所有类型的对象包括null,同时提供了方法来改变List容量的大小。

## 特征:
-   可自动扩容，和vector相比是非线程安全的。
-   可以通过List list = Collections.synchronizedList(new ArrayList(...));生成线程安全的List对象
-   ArrayList实现了Iterator接口可以通过iterator()方法获取到迭代器
-   由于ArrayList是非线程安全的，所以如果同时它进行遍历和该变它的内部接口会throw ConcurrentModificationException。（可以通过iterator解决这个问题）

## 实现结构图
![](https://i.imgur.com/6G5EC5r.png)

## 具体底层代码实现
### 定义的一些常量
```
  /**
   * jdk1.8中ArrayList的默认容量大小为10
   */
  private static final int DEFAULT_CAPACITY = 10;
  
  /**
   * 当ArrayList初始化为0时默认赋值为该空对象数组
   */
  private static final Object[] EMPTY_ELEMENTDATA = {};
  
   /**
   * 当ArrayList初始化时不指定大小时侯使用该对象数组进行初始化，同时用来区分初始化为0的那种情况
   */
  private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
  
  /**
   * 存储ArrayList元素的数组缓冲区。ArrayList元素的底层存储都是基于该数组
   */
  transient Object[] elementData;

 /**
   * 用来记录ArrayList中元素的个数
   */
  private int size;
```
###  常用方法的底层实现
#### 默认构造函数
```
   /**
    * 初始化一个容量为10的空的ArrayList集合
    */
   public ArrayList(){
       this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
   }
```
#### 带参构造函数
```
    /**
     * 参数initialCapacity代表ArrayList指定的初始化容量的大小
     * initialCapacity>0 初始化Object[] elementData数组大小为指定的initialCapacity
     * initialCapacity=0 初始化Object[] elementData 为空数组EMPTY_ELEMENTDATA
     * initialCapacity<0 抛出容量大小不合法的异常
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }


   /**
    * 参数 任意Collection对象
    * 1 将Collection对象转换成数组
    * 2 如果转换成的数组size==0 则创建一个容量为0的ArrayList对象，否则将对象数组拷贝到elementData数组中去
    *        注意: toArray()方法返回的可能不是Object[]类型，例如Arrays.asList("asList")底层返回的就是一个String[]类型的数组，只是包装在List<String>中。，所以这边判断了elementData.getClass()的类型，如果不是对象数组将它向上转型成了对象数组。
    */
   public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
    
    
    public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        @SuppressWarnings("unchecked")
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }
    
```

#### add(E e)方法 返回值为boolean
```
    /**
     * E e代表可以添加的参数 如果实例化的时候指定了泛型类型则只能添加指定类型的元素，不然则不会进行类型检查，e可以是任意类型的元素在编译的时候会自动类型擦除当成Object来处理
     * 添加元素到集合中去
     *
     * ensureCapacityInternal(size+1) 
     * 添加一个元素的时候确保容量足够大
     *
     * elementData[size++] = e;
     * 将元素添追加到elmentData数组的最后一位
     *
     * 返回boolean为true
    */ 
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    
    
    /**
     * 确保添加的时候集合的容量足够大
     *
     * 调用ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));方法进行扩容
     *
     * 调用calculateCapacity(elementData, minCapacity)计算集合需要达到的最小的容量
     */
    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }
    
    
    /**
     *  判断elementData是否为默认容量的数组，如果是则选取默认容量和添加元素后容量中的最大值
     *  如果不是直接返回minCapacity(需要达到的最小容量)
     */
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }
    
    
    /**
     *  确保容量的具体方法
     *  
     *  modCount++; 用来记录每一次结构性的调整 modCount继承与abstractList
     *
     *  判断添加元素后的容量是否大于当前集合的容量，如果超过了当前集合的容量就调用grow(minCapacity)进行扩容
     */
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    
     /**
      * 将容量扩展为当前容量的1.5倍（利用位运算）
      * 
      * 如果扩大1.5后仍然比当前需要的容量小，就扩容至minCapcity大小
      * 如果扩大1.5后超过了MAX_ARRAY_SIZE 调用hugeCapacity(minCapacity);获取指定容量
      * 
      * 将当前数组复制给新的扩容后的数组作为新的集合
      */
     private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    
    
    /**
     *  如果需要容量大小0 抛出异常
     *  不然判断是否大于MAX_ARRAY_SIZE 如果大于则返回Inter.MAX_VALUE 没有则返回MAX_ARRAY_SIZE
     */
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }

    
```

#### add(int index, E element)方法(在指定位置插入元素，即将当前元素以及往后的元素向后移动，将元素插入该位置)
```
    /**
     * 1. 检查索引位置是否合法 
     * 2. 确保容量足够大（这部和add()方法判断一样）（这步 modCount +1 因为改变了数据结构） ensureCapacityInternal(size + 1); 
     * 3. 将index后的内容向后移动一位System.arraycopy(elementData, index, elementData, index + 1,size - index);
     * 3. 将元素添加到index的位置 elementData[index] = element;
     * 4. size++; 元素的个数加一
     *
    public void add(int index, E element) {
        
         rangeCheckForAdd(index);
         // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }

    /**
     * 判断如果索引比元素个数大或者小于0 就抛出异常索引超过边界的异常
     */
    private void rangeCheckForAdd(int index) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
    
    
    注意有两一种情况，如果index==size则表示在当前集合的最后添加一个元素，此时和add()方法效果一样。
```

#### remove(int index) 将指定索引位置的元素删除 返回值为被删除的元素
```
   /**
    *  1.检查索引位置是否合法 rangeCheck(index);
    *  2. modCount++; 数据结构被修改记录+1
    *  3. E oldValue = elementData(index);获取索引位置的元素
    *  4. 计算需要向前移动的元素的个数然后将元素向前拷贝
    *  5. 将原本最后一位的元素释放（size-1并且赋值为null）
    *  6. 返回删除的元素
    */
  
   public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }


    /**
     * 判读index是否大于等于元素个数，满足则抛出经常索引位置越界
     */
    private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    E elementData(int index) {
        return (E) elementData[index];
    }
```

#### remove(Object o) 将集合中指定元素删除 返回值为boolean
```
  /*
   * 1. 分两种情况 如果o==null 则通过elementData[index] == null查找是否存为null的元素
                     o!=null 则通过o.equals(elementData[index]查找是否存为null的元素(考虑到对象和值对于值来说就是值是否相等，对象则需要是同一个对象地址才会被删除)
   */
   public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
    
    
    /**
     * 1.modCount++ 修改了集合的数据结构
     * 2.计算需要向前移动的元素的数量
     * 3.拷贝元素
     * 4.将原本最后一位元素赋值为null
     
     * 注意 此处删除并没有检查索引是否合法 因为只有元素存在才会进行这一步索引保证了索引的合法性
     */
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }

```
#### set(int index, E element)(在容器的指定位置设置指定的值)
```
 /**
  * 1 检查索引位置是否合法
  * 2 将该位置旧的值赋值给临时的变量oldValue
  * 3 将新值存放到该索引位置
  * 4 返回被删除的值
  */

 public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }

```

- get(int index)获取指定索引位置的元素
```
   public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }
```

#### clear() 清空集合中的所有元素
```
    public void clear() {
        modCount++;

        // clear to let GC do its work
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }

```

#### addAll(Collection<? extends E> c)将集合中的所有元素添加到当前集合的末尾。
 ```
       /**
        * 1. 将目标集合转换为数组
        * 2. 确保集合长度
        * 3. 将目标数组内容拷贝到当前数组的最后
        * 4. 更新集合的容量
        * 5. 根据目标数组的容量返回boolean判断是否成功。
        */
       
       public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }

 ```
 
#### addAll(int index, Collection<? extends E> c)将目标集合的元素从指定索引位置开始添加到当前集合中，并且将原本位置的元素向后移动目标集合长度的位置
 ```
       /**
        * 1. 将目标集合转换为数组
        * 2. 确保集合长度
        * 3. 计算元素需要移动的长度，然后将原本位置的元素向后移动指定长度。
        * 4. 将目标数组复制到index位置
        * 5. 根据目标数组的容量返回boolean判断是否成功。
        *
        */
       public boolean addAll(int index, Collection<? extends E> c) {
            rangeCheckForAdd(index);
    
            Object[] a = c.toArray();
            int numNew = a.length;
            ensureCapacityInternal(size + numNew);  // Increments modCount
    
            int numMoved = size - index;
            if (numMoved > 0)
                System.arraycopy(elementData, index, elementData, index + numNew,
                                 numMoved);
    
            System.arraycopy(a, 0, elementData, index, numNew);
            size += numNew;
            return numNew != 0;
        }
 ```
 
#### removeAll(Collection<?> c)删除当前集合和目标集合中元素相等的值。
 ```
        public boolean removeAll(Collection<?> c) {
            Objects.requireNonNull(c);
            return batchRemove(c, false);
        }
        
        
        private boolean batchRemove(Collection<?> c, boolean complement) {
            final Object[] elementData = this.elementData;
            int r = 0, w = 0;
            boolean modified = false;
            try {
                for (; r < size; r++)
                    //如果当前集合中不存在目标集合中的值，就将改元素存放到当前数组的w索引处
                    if (c.contains(elementData[r]) == complement)
                        elementData[w++] = elementData[r];
            } finally {
                // Preserve behavioral compatibility with AbstractCollection,
                // even if c.contains() throws.
                
                // 当c.contains() 抛出空指针异常的时候将r后的元素附加到w位置后面
                // 在某种情况下 c.contain()可能会抛出空指针的异常，出于安全考虑，进行如下操作
                //对于空指针下面的例子就会出现
                // public static boolean doesContain(Integer i) {
                //   return array.contains(Integer.toString(i));
                //   }
                //
                //    public static void main(String[] args) {
                //    System.out.println(doesContain(null));
                //  }
                
                if (r != size) {
                    System.arraycopy(elementData, r,
                                     elementData, w,
                                     size - r);
                    w += size - r;
                }
                
                //将剩余元素清空
                if (w != size) {
                    // clear to let GC do its work
                    for (int i = w; i < size; i++)
                        elementData[i] = null;
                    modCount += size - w;
                    size = w;
                    modified = true;
                }
            }
            return modified;
        }
 ```
 
#### retainAll(Collection<?> c) 取目标集合和当前集合的交集
 ```
    // 此操作和remove相反 取相同的值 通过complement=true来控制
   
     public boolean retainAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, true);
    }
 ```
 
####  replaceAll(UnaryOperator<E> operator)将集合中所有的值替换为指定值
 ``` 
        /**
         * UnaryOperator 通过传入函数方法来进行替换值
         */
        @Override
        @SuppressWarnings("unchecked")
        public void replaceAll(UnaryOperator<E> operator) {
            Objects.requireNonNull(operator);
            final int expectedModCount = modCount;
            final int size = this.size;
            for (int i=0; modCount == expectedModCount && i < size; i++) {
                elementData[i] = operator.apply((E) elementData[i]);
            }
            if (modCount != expectedModCount) {
                throw new ConcurrentModificationException();
            }
            modCount++;
        }
        
        
        例子如下所示：
            List<Integer> list = Lists.newArrayList();
            for (int i = 0; i < 10; i++) {
                list.add(i);
            }
    
            list.replaceAll(a -> a + 10);
            System.out.println(list);
 ```