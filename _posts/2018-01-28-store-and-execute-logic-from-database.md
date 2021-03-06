---
title: Хранение и выполнение логики из базы данных
permalink: /ru/store-and-execute-logic-from-database/
categories: ['Ruby', 'Туториалы', 'Database']
tags: ['ruby', 'database']

---

Решая одну из задач по своей работе, я столкнулся с проблемой хранения и выполнения какой либо логики из базы.
Приведу конкретный пример.
Нужно показать какие либо блоки текста нашим пользователям в зависимости от свойств пользователей.
<!--more-->


Задача с виду легкая и самое просторе решение - сверстать эти блоки и написать код для показа этих блоков.
А что если мы хотим избавиться от работы программиста и позволим администратору сайта самому добавлять эти блоки 
и указывать условия показа просто "натыкивая" условия через форму создания этих блоков?

Тут задача становится чуть сложнее. Создавать блоки и хранить их в базе в принципе несложно.
Остается момент сохранения этих условий в базе, парсинг и исполнение этих условий.

Так как основным языком на котором я пишу код был Ruby, то можно было бы администратору сайта предоставить поле для 
ввода логики на Ruby коде, но с точки безопасности это не является хорошим решением.
Значит нужно написать язык или DSL для описания условий, но зачем изобретать велосипед, ведь скорее всего под такие нужды уже написаны библиотеки.
И тут я начал поиск библиотек, которые подошли бы под мои нужды.

Первой библитекой которая попалась мне на глаза была Dentaku

## Dentaku
[Dentaku](https://github.com/rubysolo/dentaku) - это синтаксический анализатор логического языка формул, который позволяет подставлять в исходное выражение свои пользовательские переменные.
Язык на котором можно писать выражения для этого анализатора представляет собой некое подобие SQL.

Давайте попробуем эту библиотеку в деле!

Возьмем конкретную задачу:

 - Показать уникальное предложение каждому десятому пользователю,
   который совершил покупку 28 января в нашем магазине

Предположим, что пользователь передает нам такие данные:

```ruby

user_params_json  = <<-JSON
{ 
  "client_id": 110,
  "last_payment_date": "2018-1-28"
}
JSON
```

Давайте напишем условия, используя библиотеку dentaku

```ruby
require 'dentaku'
require 'json'

expression = '
  client_id % 10 = 0 AND
  last_payment_date = "2018-1-28"
'

calculator = Dentaku::Calculator.new
calculator.store(JSON.parse(user_params_json))
calculator.evaluate(expression)
# => true
```

Отлично, библиотека работает!
Теперь администратор нашего сайта может просто создать уникальное предложение, вбить условия для показа через GUI и сохранить это в базу.
А нам лишь останется достать из базы уникальное предложения и сравнить условия показа с параметрами пользователя.
#### Скорость
А что на счет скорости, ведь происходит парсинг условия, создание AST дерева и только лишь потом выполняется код, да-да так работает библиотека Dentaku
Выполнение данного условия показало скорость примено равную 3000 операций в секунду
Такой вариант показался для меня медленным, поэтому я дальше пошел искать соответствующие библиотеки.


## JsonLogic

Следующей библиотекой, которую мы рассмотрим, будет [JsonLogic](http://jsonlogic.com/)
Мы будем использовать имплементацию библиотеки написанную на языке Ruby - [json_logic](https://github.com/bhgames/json-logic-ruby)

json_logic позовляет создавать и хранить условия в формате JSON, а так же позвляет вставлять в условия пользовательские переменные.

Будем решать ту же задачу

Параметры пользователя останутся такими же:

```ruby
user_params_json  = <<-JSON
{ 
  "client_id": 110,
  "last_payment_date": "2018-1-28"
}
JSON
```

Давайте напишем условия, ипользуя формат JSON

```ruby
require 'json_logic'
require 'json'

rules_json = <<-JSON
{ "and": [
  {"==": [ {"%": [{"var": "client_id"}, 10]}, 0 ]},
  {"==": [ { "var": "last_payment_date" }, "2018-1-28" ]}
] }
JSON

rules = JSON.parse(rules_json)
data = JSON.parse(user_params_json)
JSONLogic.apply(rules, data)
#=> true
```

### Скорость
Выполнение данного условия показало скорость примено равную 19600 операций в секунду

Вот эта скорость мне нравится! Думаю, эта библиотека мне подойдет.

Осталось написать GUI, чтобы администратор нашего сайта писал условия не в текстовом виде, а используя удобную форму ввода условий.
