---
layout: post
title: "Автоматное программирование"
date: 2013-09-16 08:59
comments: true
categories: [pyhon, state machines, event-driven]
---

Вместо эпиграфа приведу [цитату](http://lkml.indiana.edu/hypermail/linux/kernel/0106.2/0405.html) умного [человека](http://ru.wikipedia.org/wiki/%D0%90%D0%BB%D0%B0%D0%BD_%D0%9A%D0%BE%D0%BA%D1%81). Касющуюся, впрочем, конкреного event-driven программирования:

> «Компьютер — это конечный автомат. Потоковое программирование нужно тем, кто не умеет программировать конечные автоматы»

Так как никто не хочет быть в категории “неумех”, я сегодня постараюсь показать приём программирования условного конечного автомата. К тому же это может приблизить нас к понимаю работы таких фреймворков как, например, [Twisted](http://twistedmatrix.com/trac/) или, скажем, [Tornado](http://www.tornadoweb.org/en/stable/).

<!-- more -->

###Немного теории

Вместо того косноязычного определения что могу дать я, приведу цитату из (*только не бейте*) русской википедии:

> Автома́тное программи́рование — это парадигма программирования, при использовании которой программа или её фрагмент осмысливается как модель какого-либо формального автомата. В зависимости от конкретной задачи в автоматном программировании могут использоваться как конечные автоматы, так и автоматы более сложной структуры.

Давайте поговорим о автомате (*State Machine*). Чем он у нас определяется? 

+ Входные данные (с заранее известным возможным набором данных или алфавитом). 
+ Конечное множество состояний, которые может принимать автомат. 
+ Начальное состояние. 
+ Таблица переходов, определяющая правила перехода из состояния в состояние в зависимости от текущего состояния и входного значения.

*Если вы не прогуливали теорию автоматов в ВУЗе, как я, то вы и так это всё знаете.*


###Практика

**Задача:**

В качестве задания я решил взять простую и типичную для программирования автоматов задачу - простенький обработчик разметки.
Наш обработчик будет поддерживать только 2 типа разметки:

+ \* для *курсивного* начертания
+ \*\* для **жирного** начертания

На выходе мы должны получить форматированный HTML.

**Решение:**
 
*Определим колличество возможных состояний.*

#####Возможные состояния: 
 + Начальное состояние - `plain_text`
 + Состояние `pre_bold` - когда встретили первую `*`
 + Состояние `bold` - когда встретили вторую `*` вподряд
 + Состояние `italic` - когда после первой `*` у нас, по крайней мере, один обычный символ
 + Состояние `pre_end_dold` - когда встретили первую закрывающую `*`
 
Как вы могли заметить, нам пришлось внести два не самых очевидных состояния - `pre_bold` и `pre_end_bold`. 
Дело в том, что когда мы получаем первый символ `*`, мы не можем однозначно сказать что от нас требуется италик или болд, и требуется ли вообще менять состояние, поэтому мы вынуждены ввести некие "промежуточные" состояния (это увеличивает таблицу, но упростит наш будущий код).
 
 
 *Определим правила перехода из одного состояние в другое.* 
 
#####Таблица переходов:

	
| Input \ State|plain_text  |pre_bold    |bold          |italic        |pre_end_bold|
|--------------|------------|------------|--------------|--------------|------------|
|*             |pre_bold    |bold        |pre_end_bold  |plain_text    |plain_text  
|other symbol  |plain_text  |italic      |plain_text    |plain_text    |plain_text  

В строках у нас описан входные символы, у нас их может быть всего 2 типа: символ `*` и любой другой символ.
В столбцах у нас описаны все наши состояния. На пересечении то состояние в которое переходим.
То есть если мы находимся в состоянии `plain_text` и получаем на вход `*`, то мы переходим в состояние `pre_bold`.
А в случае, если мы в состоянии `pre_bold` и получаем на вход символ отличный от `*`, то мы переходим в состоние `italic`.

Естественно, что так же мы должны определить выходное значение каждого шага нашего автомата, но я предлагаю не затягивать и определить это непосредственно в коде.

###Кодирование
А теперь у нас самое интересное - перевод нашего решениея в код.

В первом приближении у меня получилось, следующее:

```python
class SimpleMark(object):

    PLAIN_TEXT = 'plain_text'
    PRE_BOLD = 'pre_bold'
    BOLD = 'bold'
    ITALIC = 'italic'
    PRE_END_BOLD = 'pre_end_bold'

    def __init__(self):
        self.__state = self.PLAIN_TEXT
        self.__buffer = ''

    def __handle_input(self, symbol):
        if symbol == '*':
            if self.__state == self.PLAIN_TEXT:
                self.__state = self.PRE_BOLD
            elif self.__state == self.PRE_BOLD:
                self.__state = self.BOLD
                self.__buffer = '<b>'
            elif self.__state == self.BOLD:
                self.__state = self.PRE_END_BOLD
            elif self.__state == self.ITALIC:
                self.__state = self.PLAIN_TEXT
                self.__buffer = '</i>'
            elif self.__state == self.PRE_END_BOLD:
                self.__buffer = '</b>'
                self.__state = self.PLAIN_TEXT
        else:
            if self.__state == self.PRE_BOLD:
                self.__state = self.ITALIC
                self.__buffer = '<i>' + symbol
            else:
                self.__buffer = symbol
 
```

Пока выглядит как то *не_очень*, правда?

Давайте попробуем зарефакторить наш код. 
Во-первых, куча `if, elif и else` выглядит не приятно (особенно, если в вашем любимом языке программирования есть конструкция `switch/case`). Во-вторых, было бы не плохо вынести работу каждого шага в отдельную функцию (что с точки зрения автоматов является идеологический верным).

*Вынесем работу шагов автомата в функции:*

```python
class SimpleMark(object):

    PLAIN_TEXT = 'plain_text'
    PRE_BOLD = 'pre_bold'
    BOLD = 'bold'
    ITALIC = 'italic'
    PRE_END_BOLD = 'pre_end_bold'

    def __init__(self):
        self.__state = self.PLAIN_TEXT
        self.__buffer = ''

    def __set_pre_bold(self):
        self.__state = self.PRE_BOLD

    def __set_bold(self):
        self.__state = self.BOLD
        self.__buffer = '<b>'

    def __set_pre_end_bold(self):
        self.__state = self.PRE_END_BOLD

    def __set_italic(self, symbol):
        self.__state = self.ITALIC
        self.__buffer = '<i>' + symbol

    def __set_end_italic(self):
        self.__state = self.PLAIN_TEXT
        self.__buffer = '</i>'

    def __set_end_bold(self):
        self.__buffer = '</b>'
        self.__state = self.PLAIN_TEXT

    def __handle_input(self, symbol):
        if symbol == '*':
            if self.__state == self.PLAIN_TEXT:
                self.__set_pre_bold()
            elif self.__state == self.PRE_BOLD:
                self.__set_bold()
            elif self.__state == self.BOLD:
                self.__set_pre_end_bold()
            elif self.__state == self.ITALIC:
                self.__set_end_italic()
            elif self.__state == self.PRE_END_BOLD:
                self.__set_end_bold()
        else:
            if self.__state == self.PRE_BOLD:
                self.__set_italic(symbol)
            else:
                self.__buffer = symbol

```

*А теперь воспользуемся обычным рецептом эмуляции `switch/case`:*

```python
class SimpleMark(object):

    PLAIN_TEXT = 'plain_text'
    PRE_BOLD = 'pre_bold'
    BOLD = 'bold'
    ITALIC = 'italic'
    PRE_END_BOLD = 'pre_end_bold'

    def __init__(self):
        self.__state = self.PLAIN_TEXT
        self.__buffer = ''

    def __set_pre_bold(self):
        self.__state = self.PRE_BOLD

    def __set_bold(self):
        self.__state = self.BOLD
        self.__buffer = '<b>'

    def __set_pre_end_bold(self):
        self.__state = self.PRE_END_BOLD

    def __set_italic(self, symbol):
        self.__state = self.ITALIC
        self.__buffer = '<i>' + symbol

    def __set_end_italic(self):
        self.__state = self.PLAIN_TEXT
        self.__buffer = '</i>'

    def __set_end_bold(self):
        self.__buffer = '</b>'
        self.__state = self.PLAIN_TEX

    def __update_buffer(self, symbol):
        self.__buffer = symbol

    def __handle_input(self, symbol):
        jump_table_1 = {
            self.PLAIN_TEXT: self.__set_pre_bold,
            self.PRE_BOLD: self.__set_bold,
            self.BOLD: self.__set_pre_end_bold,
            self.ITALIC: self.__set_end_italic,
            self.PRE_END_BOLD: self.__set_end_bold
        }

        jump_table_2 = {
            self.PLAIN_TEXT: self.__update_buffer,
            self.PRE_BOLD: self.__set_italic,
            self.BOLD: self.__update_buffer,
            self.ITALIC: self.__update_buffer,
            self.PRE_END_BOLD: self.__update_buffer
        }

        if symbol == '*':
            jump_table_1[self.__state]()
        else:
            jump_table_2[self.__state](symbol)

```

Помимо того что я использовал словарь что бы определить связь состояние-действие для каждого из возможных состояний, я выделил ещё одну функцию `__update_buffer`, что бы была возможность явно описать переходы для второй строчки (в коде `jump_table_2`) нашей таблицы.

Вообще, раз это автомат, то у него должен быть вутренний цикл работы. Ну так давай те же напишем его!


```python
class SimpleMark(object):

    PLAIN_TEXT = 'plain_text'
    PRE_BOLD = 'pre_bold'
    BOLD = 'bold'
    ITALIC = 'italic'
    PRE_END_BOLD = 'pre_end_bold'

    def __init__(self):
        self.__state = self.PLAIN_TEXT
        self.__buffer = ''

    @property
    def output(self):
        return self.__buffer

    def __set_pre_bold(self):
        self.__state = self.PRE_BOLD

    def __set_bold(self):
        self.__state = self.BOLD
        return '<b>'

    def __set_pre_end_bold(self):
        self.__state = self.PRE_END_BOLD

    def __set_italic(self, symbol):
        self.__state = self.ITALIC
        return '<i>' + symbol

    def __set_end_italic(self):
        self.__state = self.PLAIN_TEXT
        return '</i>'

    def __set_end_bold(self):
        self.__state = self.PLAIN_TEXT
        return '</b>'

    def __update_buffer(self, symbol):
        return symbol

    def __handle_input(self, symbol):
        jump_table_1 = {
            self.PLAIN_TEXT: self.__set_pre_bold,
            self.PRE_BOLD: self.__set_bold,
            self.BOLD: self.__set_pre_end_bold,
            self.ITALIC: self.__set_end_italic,
            self.PRE_END_BOLD: self.__set_end_bold
        }

        jump_table_2 = {
            self.PLAIN_TEXT: self.__update_buffer,
            self.PRE_BOLD: self.__set_italic,
            self.BOLD: self.__update_buffer,
            self.ITALIC: self.__update_buffer,
            self.PRE_END_BOLD: self.__update_buffer
        }

        if symbol == '*':
            self.__buffer = jump_table_1[self.__state]() or ''
        else:
            self.__buffer = jump_table_2[self.__state](symbol)

    def input(self, symbol):
        self.__handle_input(symbol)

if __name__ == '__main__':
    machine = SimpleMark()

    line = "*italic* **BOLD** plain **BOLD** *italic*"
    for x in line:
        machine.input(x)
        print machine.output

```

Изменения:

 - Добавил свойство output, которое возвращает текущее значение на выходе нашего автомата

 - Добавил метод input

 - Уменьшил колличество мест, где меняется состояние выхода нашего автомата до 2-х (ибо нефиг)

 - Теперь значения выхода не накапливается в буффере, он теперь хранит только последнее значение

Такой подход добавляет реактивность в наш обработчик. 

### Что ещё?

Если сделать интерактивный ввод, то это позволит получать результат обработки одновременно с вводом.
Так же возможно реализовать метод обработки события "удаление символа", которое будет откатывать автомат в предыдущее состояние.

На этом пока всё. Замечания, по прежнему, принимаются на shoonoise@gmail.com.