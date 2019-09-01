---
title: JDK源码学习(2)之String
tags: java
categories: jdk
---

#   简述

*   来自java.lang包

*   定义
    *   String是一个final类，既不能被继承的类；
    *   String类实现了java.io.Serializable接口，可以实现序列化；
    *   String类实现了Comparable<String>，可以用于比较大小；
    *   String类实现了 CharSequence 接口，表示是一个有序字符的序列

*   转载学习 来自[CSDN](https://blog.csdn.net/snailmann/article/details/80882719 "Java8的String源码分析")

 <!--more-->

 #  构造函数
    ```
     /** 01
	* 这是一个经常会使用的String的无参构造函数.
	* 默认将""空字符串的value赋值给实例对象的value，也是空字符
	* 相当于深拷贝了空字符串""
	*/
	public String() {
        this.value = "".value;
    }

	/** 02
	* 这是一个有参构造函数，参数为一个String对象
	* 将形参的value和hash赋值给实例对象作为初始化
	* 相当于深拷贝了一个形参String对象
	*/
    public String(String original) {
        this.value = original.value;
        this.hash = original.hash;
    }

	/** 03
	* 这是一个有参构造函数，参数为一个char字符数组
	* 虽然我不知道为什么要Arrays.copyOf去拷贝，而不直接this.value = value;
	* 意义就是通过字符数组去构建一个新的String对象
	*/
    public String(char value[]) {
        this.value = Arrays.copyOf(value, value.length);
    }

	/** 04
	* 这是一个有参构造函数，参数为char字符数组,offset(起始位置，偏移量),count(个数)
	* 作用就是在char数组的基础上，从offset位置开始计数count个，构成一个新的String的字符串
	* 意义就类似于截取count个长度的字符集合构成一个新的String对象
	*/
    public String(char value[], int offset, int count) {
        if (offset < 0) {        //如果起始位置小于0，抛异常
            throw new StringIndexOutOfBoundsException(offset);
        }
        if (count <= 0) {
            if (count < 0) {     //如果个数小于0，抛异常
                throw new StringIndexOutOfBoundsException(count);
            }
            if (offset <= value.length) {      //在count = 0的前提下，如果offset<=len，则返回""
                this.value = "".value;
                return;
            }
        }
        // Note: offset or count might be near -1>>>1.
        //如果起始位置>字符数组长度 - 个数,则无法截取到count个字符，抛异常
        if (offset > value.length - count) { 
            throw new StringIndexOutOfBoundsException(offset + count);
        }
        //重点，从offset开始，截取到offset+count位置(不包括offset+count位置)
        this.value = Arrays.copyOfRange(value, offset, offset+count); 
    }

	/** 05
	* 这是一个有参构造函数，参数为int字符数组,offset(起始位置，偏移量),count(个数)
	* 作用跟04构造函数差不多，但是传入的不是char字符数组，而是int数组。
	* 而int数组的元素则是字符对应的ASCII整数值
	* 例子：new String(new int[]{97,98,99},0,3);   output: abc
	*/
    public String(int[] codePoints, int offset, int count) {
        if (offset < 0) {
            throw new StringIndexOutOfBoundsException(offset);
        }
        if (count <= 0) {
            if (count < 0) {
                throw new StringIndexOutOfBoundsException(count);
            }
            if (offset <= codePoints.length) {
                this.value = "".value;
                return;
            }
        }
        // Note: offset or count might be near -1>>>1.
        if (offset > codePoints.length - count) {
            throw new StringIndexOutOfBoundsException(offset + count);
        }  
		//以上都是为了处理offset和count的正确性，如果有错，则抛异常
		
        final int end = offset + count;

        // Pass 1: Compute precise size of char[]
        int n = count;
        for (int i = offset; i < end; i++) {
            int c = codePoints[i];
            if (Character.isBmpCodePoint(c))
                continue;
            else if (Character.isValidCodePoint(c))
                n++;
            else throw new IllegalArgumentException(Integer.toString(c));
        }
		
		//上面关于BMP什么的，我暂时也没看懂，猜想关于验证int数据的正确性，通过上面的测试就进入下面的算法
		
        // Pass 2: Allocate and fill in char[]
        final char[] v = new char[n];

        for (int i = offset, j = 0; i < end; i++, j++) {  //从offset开始，到offset + count
            int c = codePoints[i];
            if (Character.isBmpCodePoint(c))
                v[j] = (char)c;   //将Int类型显式缩窄转换为char类型
            else
                Character.toSurrogates(c, v, j++);
        }

        this.value = v; //最后将得到的v赋值给String对象的value，完成初始化
    }


	/****这里把被标记为过时的构造函数去掉了***/

	/** 06
	* 这是一个有参构造函数，参数为byte数组,offset(起始位置，偏移量),长度，和字符编码格式
	* 就是传入一个byte数组，从offset开始截取length个长度，其字符编码格式为charsetName，如UTF-8
	* 例子：new String(bytes, 2, 3, "UTF-8");
	*/
    public String(byte bytes[], int offset, int length, String charsetName)
            throws UnsupportedEncodingException {
        if (charsetName == null)
            throw new NullPointerException("charsetName");
        checkBounds(bytes, offset, length);
        this.value = StringCoding.decode(charsetName, bytes, offset, length);
    }

	/** 07
	* 类似06
	*/
    public String(byte bytes[], int offset, int length, Charset charset) {
        if (charset == null)
            throw new NullPointerException("charset");
        checkBounds(bytes, offset, length);
        this.value =  StringCoding.decode(charset, bytes, offset, length);
    }
	
	/** 08
	* 这是一个有参构造函数，参数为byte数组和字符集编码
	* 用charsetName的方式构建byte数组成一个String对象
	*/
    public String(byte bytes[], String charsetName)
            throws UnsupportedEncodingException {
        this(bytes, 0, bytes.length, charsetName);
    }

	/** 09
	* 类似08
	*/
    public String(byte bytes[], Charset charset) {
        this(bytes, 0, bytes.length, charset);
    }

	/** 10
	* 这是一个有参构造函数，参数为byte数组,offset(起始位置，偏移量),length(个数)
	* 通过使用平台的默认字符集解码指定的 byte 子数组，构造一个新的 String。
	* 
	*/
    public String(byte bytes[], int offset, int length) {
        checkBounds(bytes, offset, length);
        this.value = StringCoding.decode(bytes, offset, length);
    }

	/** 11
	* 这是一个有参构造函数，参数为byte数组
	* 通过使用平台默认字符集编码解码传入的byte数组，构造成一个String对象，不需要截取
	* 
	*/
    public String(byte bytes[]) {
        this(bytes, 0, bytes.length);
    }

	/** 12
	* 有参构造函数，参数为StringBuffer类型
	* 就是将StringBuffer构建成一个新的String,比较特别的就是这个方法有synchronized锁
	* 同一时间只允许一个线程对这个buffer构建成String对象
	*/
    public String(StringBuffer buffer) {
        synchronized(buffer) {
            this.value = Arrays.copyOf(buffer.getValue(), buffer.length()); //使用拷贝的方式
        }
    }

	/** 13
	* 有参构造函数，参数为StringBuilder
	* 同12差不多，只不过是StringBuilder的版本，差别就是没有实现线程安全
	*/
    public String(StringBuilder builder) {
        this.value = Arrays.copyOf(builder.getValue(), builder.length());
    }
	
	/** 14
	* 这个构造函数比较特殊，有用的参数只有char数组value,是一个不对外公开的构造函数，没有访问修饰符
	* 加入这个share的只是为了区分于String(char[] value)方法，用于重载，功能类似于03，我也在03表示过疑惑。
	* 为什么提供这个方法呢，因为性能好，不需要拷贝。为什么不对外提供呢？因为对外提供会打破value为不变数组的限制。
	* 如果对外提供这个方法让String与外部的value产生关联，如果修改外不的value，会影响String的value。所以不能
	* 对外提供
	*/
    String(char[] value, boolean share) {
        // assert share : "unshared not supported";
        this.value = value;
    }
    ```
 #  重点方法

 #  其他

 *  扩展问题