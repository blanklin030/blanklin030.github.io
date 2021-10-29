---
title: java面试常见问题
date: 2021-10-10 10:09:55
tags:
  - java
categories:
  - interview
---

### 反射


### 泛型
+ 目的：参数化类型
+ 用法：泛型类、泛型接口、泛型方法
  + 泛型类
  ```
  // 简单范型
  class Test<T> {
    private T var;
    public T getVar() {
      return var;
    }
    public void setVar(T var) {
      this.var = var;
    }
  } 
  // 多元泛型
  class TestKV<K,V> {
    private K key;
    private V value;
    public K getKey() {
      return key;
    }
    public void setKey(K key) {
      this.key = key;
    }
    public V getValue() {
      return value;
    }
    public void setValue(V value) {
      this.value = value;
    }
  }

  public class Demo {
    public static void main(String[] args) {
      Test<String> test = new Test<String>();
      test.setVar("aa");
      System.out.println(test.getVar());

      TestKV<String, String> testKV = new TestKV<String, String>();
      testKV.setKey("hello");
      testKV.setValue("world");
      System.out.println(testKV.getKey() +":"+testKV.getValue());
    }
  }
  ```
  + 泛型接口
  ```
  interface Test<T> {
    public T getVar();
  }
  class TestImpl<T> implement Test<T> {
    private T var;
    public TEstImpl(T var) {
      this.var = var;
    }
    public T getVar() {
      return var;
    }
    public void setVar(T var) {
      this.var = var;
    }
  }
  public class Demo{
    public static void main(String[] args) {
      Test<String> test = new TestImpl<String>("tom");
      System.out.pringln(test.getVar());
    }
  }
  ```
  + 泛型方法
  ```
  public class Demo {
    public <T> T getObject(Class<T> c) {
      T t = c.newInstance();
      retrun t;
    }
  }
  ```


### 注解


