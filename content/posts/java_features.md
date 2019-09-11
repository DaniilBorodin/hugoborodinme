+++
title = "Java Features"
date = "2019-08-28"
author = "Daniil Borodin"
cover = ""
tags = ["java"]
keywords = ["", ""]
description = "Полезные вещи в Java"
showFullContent = false
+++

# Полезные в автотестировании вещи из разных версий Java

В разных версия Java от 8 до 12 добавлялись полезные фичи. К сожалению, на многих проектах в которых я участвовал , используют только Java 8. 

## 1. List.of()

В Java 9 у List добавлен метод **.of()** который возвращает неизменяемый (immutable) список. 
До появления этого метода приходилось писать следующее:

```java
List<String> listElements = Arrays.asList("one", "two", "three");
```
Тоже самое используя метод **.of()**:
~~~java
List<String> listElements = List.of("one", "two", "three");
~~~
Кстати этот метод не позволяет добавлять в List нулевые элементы (null) выкидывая *NullPointerException*, что тоже иногда бывает удобно.

## 2. forEach()

Еще одна удобность из Java 8. Это метод **forEach()** у коллекций.
При его использовании нет не обходимости писать блоки **for**.
Как было до появления этого метода:
```java
    List<String> newListElements = new ArrayList<>();
    List<String> listElements = Arrays.asList("one", "two", "three");
    for(int i=0; i<listElements.size(); i++) {
        newListElements.add(listElements.get(i));
    }
```
Можно было сократить запись:

```java
    List<String> newListElements = new ArrayList<>();
    List<String> listElements = Arrays.asList("one", "two", "three");
    for(String element : listElements){
            newListElements.add(element);
        }
```
Но при использовании **forEach()** мы можем сделать тоже самое но намного короче:
```java
listElements.forEach(element -> newListElements.add(element));
```
И даже эту запись мы можем еще больше упростить используя формат **object :: method**
```java
listElements.forEach(newListElements::add);
```
Если для элементов списка необходимо выполнить несколько действий , то можно спользовать запись с фигурными скобками:
```java
    listElements.forEach(element -> {
            newListElements.add(element)
            //Выполняем другие действия с елементами списка
    });
```

## 3. Streams

Очень полезное и необходимое нововведение в JAVA 8. В автотестировании , где очень часто приходится работать с разного рода списками и их преобразованиями, вещь незаменимая. 

Простой пример демонстирующий удобство **Streams**:
```java
public List<String> getAllTasks(){

    List<WebElement> elementList = driver.findElements(tasksLocator);
    List<String> taskList = new ArrayList();

    for(WebElement element : elementList){
        taskList.add(element.getText());
    }
    return taskList;
}
```
 Раньше приходилось писать отдельный **for** для заполнения нового списка элементами.
 Тот же самый пример с использованием **Streams**:
```java
public List<String> getAllTasks(){

    return driver.findElements(tasksLocator)
            .stream()
            .map(WebElement::getText)
            .collect(Collectors.toList());
}
```