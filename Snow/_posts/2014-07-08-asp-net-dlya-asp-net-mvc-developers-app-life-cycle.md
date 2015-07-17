---
layout: post
title: Asp .Net для Asp .Net MVC разработчиков: Часть 1 Цикл жизни Asp .Net приложения
categories: Asp-Net
published: draft
---

**Информация в статье актуальна для Asp .Net 4.6**

##Введение##

Платформа Asp .Net изначально была построена для использования с Web Forms приложениями, а поддержка Asp .Net MVC и WebApi была добавлена в нее позже. Сама по себе платформа самодостаточна и имеет множество полезных возможностей, но большинство современных разработчиков Asp .Net MVC не имеют опыта разработки на Web Forms и поэтому не знают о этих возможностях, хотя MVC фреймворк и использует их, просто неявно для для пользователя.

В этой серии статей я хочу рассказать о строении платформы и возможностях, которые она предоставляет. Статьи будут интересны всем разработчикам Asp .Net MVC кто хочет глубже узнать о том фундаменте на котором построен MVC фреймворк.

Основа для статей взята из книги Адама Фримена [Pro Asp Net MVC Platform](http://www.apress.com/9781430265412)

##Цикл жизни приложения Asp .Net##

Вообще цикл жизни приложения - это все события приложения от его старта, до его завершения.

Цикл жизни приложения начинается тогда, когда получен первый запрос любого ресурса нашего приложения и заканчивается, когда приложение выгружается из памяти.

##Стадии цикла жизни приложения Asp .Net##

###Получен первый запрос ресурса нашего Asp .Net приложения###

1. Когда получен первый запрос любого ресурса приложения, экземпляр класса [ApplicationManager](https://msdn.microsoft.com/en-us/library/system.web.hosting.applicationmanager) создает Домен приложений, для запуска нашего приложения.

2. В пределах этого домена приложений, создается инстанс класса [HostingEnvironment](https://msdn.microsoft.com/en-us/library/system.web.hosting.hostingenvironment), который предоставляет доступ к информации о нашем приложении, такой как имя и папка, где приложение находится.

![Получен первый запрос ресурса нашего Asp .Net приложения](/images/2014-07-08-asp-net-dlya-asp-net-mvc-developers-app-life-cycle/app-manager.gif)

###Созданы объекты ядра (происходит для каждого запроса)###

##Глобальный объект приложения##
Глобальным объектом приложения является файл Global.asax, который находится в корне вашего веб-приложения. Этот файл используется во всех типах asp .net приложений (Web Forms, MVC, WebApi и т.д.).

Физически он состоит из двух файлов:

- Global.asax - не представляет практического интереса, потому что, не требует внесения каких-либо изменений с нашей стороны.

- Global.asax.cs - код.

Вот типичный код из автоматически сгенерированного Global.asax.cs для MVC приложения:
	
	namespace WebApplication
	{
	    public class MvcApplication : System.Web.HttpApplication
	    {
	        protected void Application_Start()
	        {
	            AreaRegistration.RegisterAllAreas();
	            RouteConfig.RegisterRoutes(RouteTable.Routes);
	        }
	    }
	}

MvcApplication - класс нашего приложения.

HttpApplication - главный класс Asp .Net приложения - от него нужно наследоваться, чтобы создать класс нашего приложения. От него будут наследоваться классы и MVC и WebApi и WebForms приложений.

Application_Start() - метод вызывающийся во время старта нашего приложения

##Ссылки##

[ASP.NET Application Life Cycle Overview for IIS 5.0 and 6.0](https://msdn.microsoft.com/en-us/library/ms178473)
[ASP.NET Application Life Cycle Overview for IIS 7.0](https://msdn.microsoft.com/en-us/library/bb470252)
[RequestNotification Enumeration](https://msdn.microsoft.com/en-us/library/system.web.requestnotification)
[HttpApplication instances](http://blog.andreloker.de/post/2008/05/HttpApplication-instances.aspx)