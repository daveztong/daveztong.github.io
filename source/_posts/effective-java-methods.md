---
title: effective-java-methods
date: 2016-06-22 14:34:20
categories: java
tags: [effective, java]
---

# Methods

“usability, robustness, and flexibility.”

摘录来自: Joshua Bloch. “Effective Java (Jason Arnold's Library)”。 iBooks. 

## 检查参数有效性(check validity of params)

早检查早发现早避免问题。Fail fast!

未检查参数有效性可能导致的问题:
1. 在执行过程中失败并抛出掩盖问题的异常。
1. 正常返回但是返回的是错误的结果。
1. 同样是正常返回，但是埋下了祸根，可能产生了其他状态有问题的数据，导致在不定时的将来产生不可预知的问题。

## 抛出异常
对需要抛出异常的public方法，应该使用@throws说明。常见的如:`IllegalArgumentException`,`NPE`,`IndexOutOfBoundsException` etc. 

对于需要抛出异常的方法，应该使用javadoc尽量说明抛出该异常的具体条件，并且抛出异常时应该抛出具体的异常而不是一个父类，如避免抛出exception or throwable,让使用该方法的人知道如何合理有效的使用你提供的方法。

对于nonpublic类型的方法，是由维护人员控制的，所以维护人员应该保证传入的参数都是有效的，只需要做assertion判断就行，没必要再检查然后抛出异常。

## 失败原子性(failure atomicity)

一般来说，一个失败的方法调用如果将处理的目标对象还原为调用方法之前的状态，就能称为是失败原子性的。
通常达到这个效果的做法:
* 使用immutable object.
* 针对mutable object，常用方法就是检查参数的有效性，fail before modification!

Talking is cheap, time to show some code!

```java
   /**
     * Deletes the component at the specified index. Each component in
     * this vector with an index greater or equal to the specified
     * {@code index} is shifted downward to have an index one
     * smaller than the value it had previously. The size of this vector
     * is decreased by {@code 1}.
     *
     * <p>The index must be a value greater than or equal to {@code 0}
     * and less than the current size of the vector.
     *
     * <p>This method is identical in functionality to the {@link #remove(int)}
     * method (which is part of the {@link List} interface).  Note that the
     * {@code remove} method returns the old value that was stored at the
     * specified position.
     *
     * @param      index   the index of the object to remove
     * @throws ArrayIndexOutOfBoundsException if the index is out of range
     *         ({@code index < 0 || index >= size()})
     */
    public synchronized void removeElementAt(int index) {
        modCount++;
        if (index >= elementCount) {
            throw new ArrayIndexOutOfBoundsException(index + " >= " +
                                                     elementCount);
        }
        else if (index < 0) {
            throw new ArrayIndexOutOfBoundsException(index);
        }
        int j = elementCount - index - 1;
        if (j > 0) {
            System.arraycopy(elementData, index + 1, elementData, index, j);
        }
        elementCount--;
        elementData[elementCount] = null; /* to let gc do its work */
    }
```

这段代码取自Vector中。在移除元素时提前检查数组是否越界，并在javadoc中使用`@throws`说明抛出的异常和触发条件。

* 复制替换. 复制一份原始数据，对副本进行操作，顺利完成之后再替换原始数据。 如Collections.sort()在进行排序之前就是先将原始数据存在一个数组中，如果排序失败，原始数据原封不动。
* 
JDK source code:
```java
    default void sort(Comparator<? super E> c) {
        Object[] a = this.toArray();
        Arrays.sort(a, (Comparator) c);
        ListIterator<E> i = this.listIterator();
        for (Object e : a) {
            i.next();
            i.set((E) e);
        }
    }
```

就以上看来，方法参数限制越严密越好，其实不然，如果一个方法对传入的参数都能够优雅的处理不用太多的限制对于使用该方法的人来说无疑是一大福音。

__以上这几点看上去很简单朴实，但如果真正遵守并养成这些习惯，对代码质量的提高是可见一斑的。__

## 保护性复制(Defensive copy)
保护性复制是保护对象不变性，健壮性的一种策略。API提供者本意是想提供一个可靠稳定的对象，但往往不经意间对象的内部状态就被API的使用者所更改了，而且是无意识的。举个栗子:

```java
private class Period{
        private Date start;
        private Date end;

        /**
         * 
         * @param start
         * @param end
         *
         * @throws IllegalArgumentException if start is greater than end
         * @throws NullPointerException if start or end is null
         */
        public Period(Date start, Date end) {
            if (start.compareTo(end)>0)
                throw new IllegalArgumentException("start must less than end");

            this.start = new Date(start);
            this.end = new Date(end);
        }

        public Date getStart() {
            // return new Date(start);
            return start;
        }

        public Date getEnd() {
            // return new Date(end);
            return end;
        }
    }
```
__The invariants of period can be broken easily!__
传入start和end后，对start和end的修改会直接影响Period的内部状态,使用get方法获取到日期后也可以修改对象的内部状态,可以说这就是发布了一个不安全的对象。使用defensive copy就可以修复这个问题,在构造器和getter方法中都复制一份源对象即可。另外一种技巧也可以解决这个问题，period内部使用long来存储时间，就不用担心状态被更改。

## 谨慎设计方法签名

### 选择有意义的名字

遵守约定的命名规范，如变量名词，方法动词开头等。

### 不要过度提供工具方法

过多的方法导致对象的维护难度提高，易用性下降。特别是对于接口而言，过多的方法导致实现起来比较困难，Think before in doing anything! 如果觉得没必要就不提供。

### 避免过长的参数列表
参数尽量不要超过四个，多了使用起来容易出错，特别当有多个相同类型的参数存在的时候,如果顺序弄错了，最终结果总是不对，也很难发下bug的所在。

针对参数过多的情况常见的应对方式有:
1. 拆分成几个方法，有可能会产生很多方法。
2. 新建一个helper类包含这些参数，再将这个helper类作为参数传入。
3. 利用Builder pattern。

### 优先使用接口类型作为参数而不是实现类

这个没什么好说的.

### 使用含有两个元素的枚举代替boolean类型

使用枚举不仅可读性更强，而且更容易扩展，可以添加其他选项和做转换处理等。



