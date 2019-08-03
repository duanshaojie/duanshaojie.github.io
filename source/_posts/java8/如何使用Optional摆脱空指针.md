---
title: 如何使用Optional摆脱空指针
tags: java8
categories: java
---

##   简述

   在刚使用Java编写代码的时候，我碰到最多的错误可能就是NullPointerException,后来慢慢的开始使用if-else来规避，
但是大量的if-else真的很难看，可读性也非常差。而Java 8引入的这个新类java.util.Optional<T>可以使用简单的代码解决这个问题，就记录下学习的过程吧。

<!--more-->

##  Optional的常用方法

   这里主要使用源码来解读Optional，如下
   ```
/**
 * 这是一个包含null或者非null值得容器，当值不为空，isPresent()返回true，并且get()
 * 将返回这个值；
 *
 * 提供了取决于值是否存在的方法
 * 如orElse()，当值不存在时返回一个默认的值；
 * 如ifPresent()，当值存在执行。
 *
 * 这是一个基于价值的类，应避免使用使用对象比较（==），标识Hash或同步
 * @since 1.8
 */
public final class Optional<T> {
    /**
     * 通用实例（一个空的实例）
     */
    private static final Optional<?> EMPTY = new Optional<>();

    /**
     * Optional的值
     */
    private final T value;

    /**
     * 构造器，一个空值
     */
    private Optional() {
        this.value = null;
    }

    /**
     * 返回一个空值得实例
     */
    public static<T> Optional<T> empty() {
        @SuppressWarnings("unchecked")
        Optional<T> t = (Optional<T>) EMPTY;
        return t;
    }

    /**
     * 默认值得Optional
     * 如果传入值为空则抛出NullPointerException异常
     */
    private Optional(T value) {
        this.value = Objects.requireNonNull(value);
    }

    public static <T> Optional<T> of(T value) {
        return new Optional<>(value);
    }

    /**
     * 如果为空则返回空Optional
     */
    public static <T> Optional<T> ofNullable(T value) {
        return value == null ? empty() : of(value);
    }

    /**
     * 获取Optional值，若为空则抛异常
     */
    public T get() {
        if (value == null) {
            throw new NoSuchElementException("No value present");
        }
        return value;
    }

    /**
     * Optional 值是否为空呢
     */
    public boolean isPresent() {
        return value != null;
    }

    /**
     * 如果值不为空，则执行consumer
     */
    public void ifPresent(Consumer<? super T> consumer) {
        if (value != null)
            consumer.accept(value);
    }

    /**
     * 如果值不为空，并且匹配predicate，则返回这个值
     */
    public Optional<T> filter(Predicate<? super T> predicate) {
        Objects.requireNonNull(predicate);
        if (!isPresent())
            return this;
        else
            return predicate.test(value) ? this : empty();
    }

    /**
     * 如果值存在，则将映射函数应用于这个值，否则返回Optional
     * <pre>{@code
     *     Optional<FileInputStream> fis =
     *         names.stream().filter(name -> !isProcessedYet(name))
     *                       .findFirst()
     *                       .map(name -> new FileInputStream(name));
     * }</pre>
     */
    public<U> Optional<U> map(Function<? super T, ? extends U> mapper) {
        Objects.requireNonNull(mapper);
        if (!isPresent())
            return empty();
        else {
            return Optional.ofNullable(mapper.apply(value));
        }
    }

    /**
     * 如果值存在，将提供的Optional-bearing映射于该函数，否则返回空Optional；
     * 类似于Map()，但映射器是一个Optional结果
     */
    public<U> Optional<U> flatMap(Function<? super T, Optional<U>> mapper) {
        Objects.requireNonNull(mapper);
        if (!isPresent())
            return empty();
        else {
            return Objects.requireNonNull(mapper.apply(value));
        }
    }

    /**
     * 如果值存在则返回它，否则返回写入的值
     * 无论值是否存在都会执行
     */
    public T orElse(T other) {
        return value != null ? value : other;
    }

    /**
     * 如果存在返回值，否则返回other的值
     * 只有值不存在时才会执行
     */
    public T orElseGet(Supplier<? extends T> other) {
        return value != null ? value : other.get();
    }

    /**
     * 如果值不为空，返回这个值，否则抛出异常
     */
    public <X extends Throwable> T orElseThrow(Supplier<? extends X> exceptionSupplier) throws X {
        if (value != null) {
            return value;
        } else {
            throw exceptionSupplier.get();
        }
    }

    /**
     * 值比较
     */
    @Override
    public boolean equals(Object obj) {
        if (this == obj) {
            return true;
        }

        if (!(obj instanceof Optional)) {
            return false;
        }

        Optional<?> other = (Optional<?>) obj;
        return Objects.equals(value, other.value);
    }

    /**
     * 值hash
     */
    @Override
    public int hashCode() {
        return Objects.hashCode(value);
    }

    @Override
    public String toString() {
        return value != null
            ? String.format("Optional[%s]", value)
            : "Optional.empty";
    }
}

   ```

##  Optional的使用
    针对上面源码的方法，总结一些使用。本人算是初次使用，如有错误还望指正交流~~

*   在下面的案例中，由于没有“James”的用户信息，直接使用userEntity会抛出java.lang.NullPointerException异常。
 所以你在使用的时候，需要增加if(null != userEntity)。
```
   public static void main(String[] args) {
       UserEntity userEntity = queryUserEntityByName("James");
       System.out.println(userEntity.getName());
   }

   private static UserEntity queryUserEntityByName(String name) {
       Map<String, UserEntity> userEntityMap = new HashMap<>(1);
       userEntityMap.put("edison", new UserEntity("edison", 2L));
       return userEntityMap.get(name);
   }
```

*   而使用了Optional的代码如下，因为返回的是Optional，该对象是一个包含空或非空的容器。
下面如果直接使用userOptional.get(),那么当“Jame”不存在的时候就会抛出异常，所以我们给userOptional增加一个
默认的UserEntity
```
    public static void main(String[] args) {
        Optional<UserEntity> userOptional = queryUserEntityByName("James");
        System.out.println(userOptional.orElse(new UserEntity()).getName());
    }

    private static Optional queryUserEntityByName(String name) {
        Map<String, UserEntity> userEntityMap = new HashMap<>(1);
        userEntityMap.put("edison", new UserEntity("edison", 2L));
        return Optional.ofNullable(userEntityMap.get(name));
    }
```

*   上面的案例还可以使用如下来避免空指针,orElseGet()与orElse()的区别在于，orElse()无论值是否为空都会执行一遍；
而orElseGet()只有当值为空时，才会执行
```
System.out.println(userOptional.orElseGet(() -> new UserEntity()).getName());
```

*   map()与flatMap():看源码我们就可以知道，两者没有什么区别，唯一的区别就是两者的入参不一样，那又如何使用呢？
总的来说，这里的map与Stream中的map还是十分类似的，都是链路思维。
```
//map-u.getName()返回的是String
Optional<UserEntity> userOptional = queryUserEntityByName("edison");
System.out.println(userOptional.map(u -> u.getName()).get());

public class UserEntity {

    ...

    public String getName(){
        return this.name;
    }
}

//而flatMap的u.getName()返回的是Optional
Optional<UserEntity> userOptional = queryUserEntityByName("edison");
System.out.println(userOptional.flatMap(u -> u.getName()).get());


public class UserEntity {

    ...

    public Optional getName(){
        return Optional.ofNullable(this.name);
    }
}
```

