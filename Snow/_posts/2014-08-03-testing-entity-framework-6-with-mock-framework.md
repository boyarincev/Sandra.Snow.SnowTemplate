---
layout: post
title: Тестирование Entity Framework 6 с помощью mock фреймворка
categories: entity-framework
published: draft
---
**Эта статья актуальна для Entity Framework 6** для тестирования EF5 и более ранних смотрите [Testing with a Fake Context](http://romiller.com/2012/02/14/testing-with-a-fake-dbcontext/).

Когда мы пишем тесты, часто хочется избежать использования базы данных. Entity Framework позволяет проводить тестирование используя данные находящиеся в памяти (in-memory), а не в базе данных.

##Варианты создания тестовых дублей##
Есть два подхода для проведения такого типа тестирования:

- **Создание ваших собственных тестовых дублей** - этот подход подразумевает создание ваших собственных реализацией (имплементаций) контекста и DbSet'ов. Вы полностью контролируете как ведут себя эти классы, но это потребует от вас написание некоторого количества кода. Подробнее рассматривается в [этой](https://msdn.microsoft.com/ru-ru/data/dn314431) статье.
- **Использование mocking framework для создание тестовых дублей** - используя mocking framework (такого как [Moq](http://www.moqthis.com/)) вы можете иметь in-memory имплементации контекста и Dbset'ов созданных динамически во время работы приложения.

Для демонстрации использования mocking framework мы будем использовать Moq. Проще всего получить Moq - установить [Moq пакет для NuGet](http://www.nuget.org/packages/Moq/).

##Ограничения возникающие при тестировании EF используя данные находящиеся в памяти, а не в базе данных##
При проведении такого тестирования мы используем LINQ to Object, чтобы запрашивать данные, тогда как при использовании базы данных используется LINQ to Entities  в результате чего запросы транслируются в SQL запросы.

Примером является загрузка связанных данных. Допустим мы имеем набор Блогов, имеющих Записи. Когда мы запрашиваем Блог, он уже будет содержать загруженные посты. Однако при использовании базы данных, они могут быть не загружены.

По этой причине, кроме unit тестирования, рекомендуется всегда включить и end-to-end тестирование, чтобы гарантировать, что приложение корректно работает с базой данных.
