---
layout: post
title: ASP.NET 5 и AngularJS Часть 1, Настраиваем Grunt и AngularJS
categories: Asp-Net-5
published: draft
---

Это первая часть в серии постов по построению приложения на Asp .Net 5 (Asp .Net vNext) с использованием AngularJS. В этой серии постов, я покажу как создать простое веб-приложение целью которого является отслеживание ежедневного следования выбранных привычек, используя Asp .Net 5 и AngularJS.

Основной функционал приложения сводится к ежедневному контролю, следования выбранной привычке.

Вы можете скачать код обсуждаемый в этой статье из GitHub:

https://github.com/boyarincev/GetHabitsAspNet5

В этой статье я опишу, как подготовить Asp .Net 5 проект, в частности, как настроить Grunt на минификацию и объединение javascript файлов автоматически.

##Содаем Asp .Net 5 проект##

Мы будем использовать [Visual Studio 2015 Community Edition](https://www.visualstudio.com/downloads/download-visual-studio-vs) - она бесплатна для личного использования.

После того, как вы установите Visual Studio 2015, выберите File, New, Project, затем в разделе Templates, Visual C#, Web выберите ASP.NET Web Application и назовите новый проект "GetHabitsAspNet5"

![Создаем новый Asp.NET 5 проект](/images/asp-net-5/part1/create-new-project.jpeg)

Выберите, Empty шаблон в разделе ASP.NET 5 Preview Templates

![Выбираем Empty Template](/images/asp-net-5/part1/choose-empty-template.jpeg)

После того, как студия закончит генерацию шаблона, вы получите пустой ASP.NET 5 проект.

![Empty шаблон в Solution Explorer](/images/asp-net-5/part1/empty-project-sol-explorer.jpeg)

Шаблон ASP.NET 5 значительно отличается от предыдущих версий. Теперь солюшен разделен на две папки, первая "Solution Items", вторая "src". Папка src содержит сам проект GetHabitsAspNet5.

GetHabitsAspNet5 проект содержит специальную папку wwwroot. Назначение этой папки - хранить весь итоговый контент сайта: html, javascript, css файлы, картинки и другие ресурсы. Не следует хранить в ней файлы с исходным кодом, Less файлы, файлы javascript, которые еще планируется объединять и минифицировать.

Давайте в корне проекта GetHabitsAspNet5 создадим папку Scripts - в ней мы будем хранить все javascript файлы.

![Создаем папку Scripts](/images/asp-net-5/part1/script-folder.jpeg)

Вероятно раньше, для объединения и минификации javascript вы использовали механизм бандлов из ASP.NET MVC или использовали функции расширения WebEssentials, но теперь, с нативной поддержкой Grunt, мы можем всю подготовку фронтенда делать с помощью него.

##Используем NPM для получения необходимых пакетов##

Visual Studio 2015 нативно поддерживает три пакетных менеджера: NuGet, NPM, Bower.

