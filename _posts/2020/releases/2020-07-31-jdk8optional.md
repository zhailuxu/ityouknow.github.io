---
layout: post
title: 基于JDK8中Optional写出可读性高的代码
category: java
excerpt: JDK8中引入了函数式编程，大大提高了我们编写代码的可读性，其中Optional则是为了避免NPE而生，下面我们就来看看它是如何提高代码可读性的。
tags: [java]
---

# 一、前言
JDK8中引入了函数式编程，大大提高了我们编写代码的可读性，其中Optional则是为了避免NPE而生，下面我们就来看看它是如何提高代码可读性的。


# 二、Optional 使用
假设我们代码里面下面DO对象：
```Java
 static class Wheel {
    public String getBrand() {
        return brand;
    }

    public void setBrand(String brand) {
        this.brand = brand;
    }

    private String brand;
}
```

```Java
 static class Car {
    public Wheel getWheel() {
        return wheel;
    }

    public void setWheel(Wheel wheel) {
        this.wheel = wheel;
    }
    private Wheel wheel;
}

```

```Java
 static class Person {
    public Car getCar() {
        return Car;
    }

    public void setCar(Car Car) {
        this.Car = Car;
    }

    private Car Car;
}
```


在不用Optional时候，如果我们想获取Person内嵌对象Wheel中的brand属性变量的值，在考虑避免NPE的情况下，代码可能如下：

```Java
String brand = null;
if (null != person){
    if (null != person.getCar()){
        if (null != person.getCar().getWheel()){
             brand = person.getCar().getWheel().getBrand();
        }
    }
}
```

如上是典型的箭头型代码，写起来比较琐碎，并且可读性不是很高。下面看使用Optional改造后的代码：

```Java
 brand = Optional.ofNullable(person) //转换为Optional进行包裹
        .map(p -> p.getCar()) //获取Car对象
        .map(car -> car.getWheel()) //获取Wheel对象
        .map(wheel -> wheel.getBrand()) //获取brand
        .orElse("玛莎拉蒂"); //如果中间有对象为null，则返回默认值"玛莎拉蒂"
```

如上代码，经过改造后，代码的可读性得到了提高，而写代码的成本却大大降低。上面使用Optional后，无论person为null，或者其内部的car为null，或者wheel对象为null，都不会出现NPE，而是会返回默认的“玛莎拉蒂”。


# 三、总结
善用工具，可以解放生产力，提高代码可读性，提高代码稳定性，何乐而不为那？最后，之前然也要知其所以然，Optional内部如何实现的那？大家可以翻看其代码看看，其实很简单。
