---
layout: post
author: kormang
---

In the next few articles I'll be talking about complexity of code, how do we get into situations where our code looks like spaghetti, and how do we get out of those situations. Later different subject may come, first of which will be asynchronous and concurrent programming.

The goal is to explain some of the most important things I have learned from my experience and the experience of others (contained in books, articles, in-person talks, etc.), summarize all that as much as possible without losing the point, and hopefully help people around me write better software, and make their, and my own too, lives easier.

Examples will be predominantly in **JavaScript**, **TypeScript** and **Python** to be accessible to as large an audience as possible. Sometimes other languages will be used to better illustrate the concepts. Hopefully, examples in those other languages will be useful for better understanding of the concepts. Knowledge of basic JavaScript is absolutely required to follow the material. Knowledge of **Java** is not required as Java examples should be simple and intuitive enough to allow anyone to understand them. Examples in **C/C++** can be skipped, but they do present additional details to better understand the subject. Also, this material is initially and primarily meant for my colleagues, and in the company I work for at the moment of starting this journey.

It is hard to explain anything without examples, and it is hard to make examples that are short, yet illustrate the point perfectly, at least when subject is complexity of code. Reading examples will require focus; it is not expected that the reader can skim through the examples.

I'll be using the term "unit of code" to represent a function, class, file, component, module, basically any level of code organization.

Some concepts might seem too obvious, and you will have many "Well, I know this already" moments. That might be true, but it can also be the "Illusion of Competence" (a synonym for the Dunning-Kruger effect).

> _"All the advice that has come out of our revolution does not help much until it becomes second nature"_ - Apprenticeship Patterns, Guidance for the Aspiring Software Craftsman, by David H. Hoover and Adewale Oshineye.

I don't claim that everything I write here is absolute truth, and I would definitely like to get some feedback and grow from it. However, I would be writing this if I don't think many of you could find it useful, but for many of these advices to become useful, practice is needed.

If you don't practice until some of the advice presented here becomes natural to you, it will be painful to follow these pieces of advice. We can compare it to swimming. Let's say you know how to swim, but you get tired easily or swim too slowly. You could learn a new technique, you could learn to submerge your head underwater. But to do that, you need to let go of the old way of swimming, and for some time, until you get good at it, you need to accept that you'll swim worse (slower, more awkwardly) than you did previously by swimming the old way. The same can be said about skiing. You might have learned to ski in some suboptimal way. Maybe you can swing your hips during turns, and you're able to ski all the slopes you want. However, you might be struggling and losing too much energy. To overcome that problem, you need to learn to carve, and to do that you need to let go of the old way of skiing, to reduce your speed, and to pick easier slopes. Be it swimming, skiing, or some other activity, switching to a new technique will feel awkward, strange, and unnatural at the beginning. However, after you successfully master the new technique, it will definitely pay off. That period of decreased performance is the price of growth.

In the process of learning new things, you will probably overuse some of the advice, trying to apply it even when it is not applicable. That is OK. That is part of learning, of exploration. You will then understand the limitations of each piece of advice, each pattern, each technique, and each approach to solving software engineering problems. To make that process easier and to create fewer frustrations among your colleagues, remember that each piece of advice presented here is not always applicable, and sometimes you should do exactly the opposite. Use judgment each time you try to apply it - can this advice be applied here, depending on the goals and circumstances? Also note that at the beginning, you will be more likely to conclude that it is not applicable, but that is because you haven't mastered these pieces of advice yet.

Stay tuned...
