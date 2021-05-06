---
title: Python class method extensions
date: 2021-05-05T21:00:00+01:00
draft: false
tags: 
    - python
category: python
keywords: 
    - python
---

One of the things I've always envied from Swift developers is [how good class extension support is there](https://docs.swift.org/swift-book/LanguageGuide/Extensions.html). Although this is not unique to Swift (C# and JavaScript had it earlier), it's the one I know most about since it's iOS-related.

I've had to work with Python code for a very long time now, and the lack of a simple 'extension' keyword is painful. Yes, this is not technically needed in Python as you can dynamically add attributes to classes at runtime, but it makes the process somewhat more cumbersome and way less intuitive. These are the two main ways of doing it I'd like to list:

## Using `setattr`

The [`setattr` method](https://docs.python.org/3/library/functions.html#setattr) lets you programmatically add an attribute to the provided object - which lets you add methods or variables to it. Take a simple class for instance:

```python
class Rectangle():
    def __init__(self):
        self._length = 5
        self._width = 3

    def perimeter():
        return 2*(self._length + self._width)
```

Imagine this class has been defined by a third-party module, and that you want to compute the area of the rectangle. While you could write your own method that retrieves all sides and computes it, it would be cool if you could just do `rectangle.area()`. It seems the original developer forgot to include such basic functionality, but we can add the method programmatically:

```python
def area(self):
    return self._length * self._width

setattr(Rectangle, 'area', area)
```

Notice how the `area` method can have a `self` parameter even though it is defined outside the class. The `value` of self will obviously be initialized at runtime once there's an instance of the Rectangle class.

## Using Class inheritance

In Python you can define a child class of a superclass like this:

```python
class Person():
    def __init__(self, name, surname):
        self.name = name
        self.surname = surname


class Student(Person):
    def __init__(self, name, surname, student_id):
        super().__init__(self, name, surname)
        self.student_id = student_id
```

The cool thing about Python is that the child class can have the same name as the parent one, and that will define a new class that replaces the old one, but is also derived from it. Imagine trying this in Java.

So, following my previous example we could do:

```python
# File mymodule.rectangle.py
class Rectangle():
    def __init__(self):
        self._length = 5
        self._width = 3

    def hypotenuse(self):
        return 2*(self._length + self._width)

# File mymodule.newclass.py
import mymodule.rectangle

class Rectangle(mymodule.rectangle.Rectangle):
    def area(self):
        return self._length * self._width
```
And this will work as good as the previous option.

## Pros and cons of each

Using `setattr` has the disadvantage of IDEs like PyCharm not really understanding what is going on, since the additional methods won't be added to the class until you reach runtime, of course - so you'll have to deal with that. Using class inheritance fixes this, but can lead to convoluted or very poorly-written patterns where you end up with a mess of different classes with the same name, but different import path and functionalities.

Consider each option with caution.
