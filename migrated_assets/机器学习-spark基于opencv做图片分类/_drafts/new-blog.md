---
title: new-blog
tags:
mathjax: true
---


**Marxico** is a delicate Markdown editor for Evernote. With reliable storage and sync powered by Evernote, **Marxico** offers greate writing experience. 

- **Versatile** - supporting code highlight, *LaTeX* & flow charts,  inserting images & attachments by all means.
- **Exquisite** -  neat but powerful editor, featuring offline docs, live preview, and offering the [desktop client][1] and offline [Chrome App][2].
- **Sophisticated** - deeply integrated with Evernote, supporting notebook & tags, two-way bind editing.   

----------


# Introducing Markdown

> Markdown is a plain text formatting syntax designed to be converted to HTML. Markdown is popularly used as format for readme files, ... or in text editors for the quick creation of rich text documents.  - [Wikipedia](http://en.wikipedia.org/wiki/Markdown)

As showed in this manual, it uses hash(#) to identify headings, emphasizes some text to be **bold** or *italic*. You can insert a [link](http://www.example.com) , or a footnote[^demo]. Serveral advanced syntax are listed below, please press `Cmd + /` to view Markdown cheatsheet.

## Code block
``` python
@requires_authorization
def somefunc(param1='', param2=0):
    '''A docstring'''
    if param1 > param2: # interesting
        print 'Greater'
    return (param2 - param1 + 1) or None
class SomeClass:
    pass·
>>> message = '''interpreter
... prompt'''
```

## math jax
$$	x = \dfrac{-b \pm \sqrt{b^2 - 4ac}}{2a} $$

This equation  $\cos^2{\theta} = \cos^2 \theta - \sin^2 \theta =  2 \cos^2 \theta - 1$  is inline.

$$
\begin{aligned}
\dot{x} & = \sigma(y-x) \\
\dot{y} & = \rho x - y - xz \\
\dot{z} & = -\beta z + xy
\end{aligned}
$$

$$
\frac{\partial u}{\partial t} = h^2 \left( \frac{\partial^2 u}{\partial x^2} + \frac{\partial^2 u}{\partial y^2} + \frac{\partial^2 u}{\partial z^2}\right)
$$

$$\frac{|ax + by + c|}{\sqrt{a^{2}+b^{2}}} $$


$$\begin{equation}\label{eq1}
e=mc^2
\end{equation}$$

$$\begin{equation}\label{eq2}
\begin{aligned}
a &= b + c \\
  &= d + e + f + g \\
  &= h + i
\end{aligned}
\end{equation}$$

$$\begin{align}
a &= b + c \label{eq33} \\
x &= yz \label{eq4}\\
l &= m - n \label{eq5}
\end{align}$$

$$\begin{equation}\label{eq6}
\min \limits_{p*, q*} \sum\limits_{r_{u, i}\ is\ knwon}(r_{ui} -p_u^Tq_i)^2 + \lambda(||p_u||^2 + ||q_i||^2) \end{equation}$$

$$
PR(v) = \begin{cases}
\alpha\sum_{v‘ \in in(v)} \frac{1}{1} \\
(1 - \alpha)
\end{cases}
$$

引用上述公式$\ref{eq1}$

## Table
| Item      |    Value | Qty  |
| :-------- | --------:| :--: |
| Computer  | 1600 USD |  5   |
| Phone     |   12 USD |  12  |
| Pipe      |    1 USD | 234  |

## Diagrams
### Flow charts
```flow
st=>start: Start
e=>end
op=>operation: My Operation
cond=>condition: Yes or No?

st->op->cond
cond(yes)->e
cond(no)->op
```
### Sequence diagrams 
```sequence
Alice->Bob: Hello Bob, how are you?
Note right of Bob: Bob thinks
Bob-->Alice: I am good thanks!
```

> **Note:** You can find more information:

> - about **Sequence diagrams** syntax [here][3],
> - about **Flow charts** syntax [here][4].

## Checkbox
You can use `- [ ]` and `- [x]` to create checkboxes, for example:

- [x] Item1
- [ ] Item2
- [ ] Item3

> **Note:** Currently it is only partially supported. You can't toggle checkboxes in Evernote. You can only modify the Markdown in Marxico to do that. Next version will fix this.  


## Dancing with Evernote

### Notebook & Tags
**Marxico** add `@(Notebook)[tag1|tag2|tag3]` syntax to select notebook and set tags for the note. After typing `@(`, the notebook list would appear, please select one from it.  

### Title
**Marxico** would adopt the first heading encountered as the note title. For example, in this manual the first line `Welcome to Marxico` is the title.

### Quick Editing
Note saved by **Marxico** in Evernote would have a red ribbon button on the top-right corner. Click it and it would bring you back to **Marxico** to edit the note. 

> **Note:** Currently **Marxico** is unable to detect and merge any modifications in Evernote by user. Please go back to **Marxico** to edit.

### Data Synchronization
While saving rich HTML content in Evernote, **Marxico** puts the Markdown text in a hidden area of the note, which makes it possible to get the original text in **Marxico** and edit it again. This is a really brilliant design because:

- it is beyond just one-way exporting HTML which other services do;
- and it avoids privacy and security problems caused by storing content in a intermediate server. 

> **Privacy Statement: All of your notes data are saved in Evernote. Marxico doesn't save any of them.** 

## Feedback & Bug Report
- Twitter: [@gock2][7]
- Email: <hustgock@gmail.com>

----------
Thank you for reading this manual. Now please press `Cmd + M` and click `Link with Evernote`. Enjoy your **Marxico** journey!


[^demo]: This is a demo footnote. Read the [MultiMarkdown Syntax Guide](https://github.com/fletcher/MultiMarkdown/wiki/MultiMarkdown-Syntax-Guide#footnotes) to learn more. Note that Evernote disables ID attributes in its notes , so `footnote` and `TOC` are not actually working. 

  [1]: http://marxi.co/client_en
  [2]: https://chrome.google.com/webstore/detail/kidnkfckhbdkfgbicccmdggmpgogehop
  [3]: http://bramp.github.io/js-sequence-diagrams/
  [4]: http://adrai.github.io/flowchart.js/
  [5]: http://dillinger.io
  [6]: http://stackedit.io
  [7]: https://twitter.com/gock2







