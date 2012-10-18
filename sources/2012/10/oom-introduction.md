Title: Object Oriented Metrics - WTF?
Tags: OOP, Metrics, Code Quality
Slug: object-oriented-metrics-introduction
Date: 2012-10-18 22:00
Category: OOP

#WTF is that?

We'll start from an image. Did you see in past something like that?

![Overview pyramid](/images/2012/10/overview-pyramid.png "An example of overview pyramid")

This is called **Overview Pyramid**. This is, I think, most commonly used visualization of **Object oriented metrics**. What are they?
**Metrics** in short, are some kind of **numeric values**, describing code (there is also a possibility to describe classess without code
using for e.g. UML), or in Object Oriented Metrics, describing **classess**. It might be easier to describe it on an example. Let's take 
simpliest metric - **LOC** (Lines Of Code), which is also seen on an overview pyramid. This metric describes how many lines of code 
are in all defined methods.

##Ok, but why do we need them?
During my career I found myself in many situations that I had to review people's code. And I have to say I'm getting really often angry 
on what I see. Code likes to be really unreadable - lot's of *if* instructions, hellish long methods, everywhere high coupling (sometimes
I think people don't know what's an DI, they just heard it it's fashionable in new frameworks but what the hell... I don't need it!).
This might be because mostly I'm used to look into PHP/JavaSript code and I think these languages are really really bad for person which
learns programming.

Additionally I saw also people using for e.g. *'Design Patterns'* without any knowledge about them. The're used only for the sake of 
using them. And this is make you more problematic to mainain the code than just have it done simple.

Personally I think that people before starting to learn about Design Patterns should read about something that is called **Design Disharmonies**.
These disharmonies are a bad thing in programming world and you should avoid them to maintain your code **readable** and **maintanable**. They
are called **bad smells**.

##Still don't know why do I need these metrics them?
Ok, so if your code grows really big, you need to use more and more time only to verify the code base. If it's really huge then detecting such a
disharmonies is really hard work. Then the metrics are handy. Using them you can detect **bad smelling code** and verify if somebody really farted
there and left some bad suprises.

##So what's with Overview Pyramid (Watch out! Yet another boring explaination)?
Let's get back to Overview Pyramid. There was one metric already described as an example: **LOC**. But we have here also other ones:
**CYCLO** - Cyclomatic Number. This number counts all possible paths that it can take. For e.g. **if** instruction
will increase this number by one, cause program can go though this if or not. Of course bigger value means higher complexity, which in many 
cases might be bad.

Further on the left side you can find **NOM** - Number Of Methods, **NOC** - Number Of Classess and **NOP** - Number Of Packages, depending
on which language it can be Package (Java), Module (Python, JavaScript), Namespace (C++, PHP) etc.
