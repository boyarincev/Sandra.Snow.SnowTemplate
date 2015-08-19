---
layout: post
title: ASP.NET 5 Routing
categories: ASP-NET-5
published: draft
---

Давайте разберемся, какие изменения произошли в системе роутинга в ASP.NET 5 по-сравнению с предыдущими версиями.

##Как была огранизована система маршутизации до ASP.NET 5##
Маршрутизация до ASP.NET 5 осуществлялась с помощью ASP.NET модуля [UrlRoutingModule](https://msdn.microsoft.com/en-us/library/system.web.routing.urlroutingmodule(v=vs.100).aspx). Модуль проходил через [коллекцию](https://msdn.microsoft.com/en-us/library/system.web.routing.routecollection(v=vs.100).aspx) маршрутов (как правило объектов класса [Route](https://msdn.microsoft.com/en-us/library/system.web.routing.route(v=vs.110).aspx)) хранящихся в статическом свойстве `Routes` класса [RouteTable](https://msdn.microsoft.com/en-us/library/system.web.routing.routetable(v=vs.110).aspx), выбирал маршрут, который подходил под текущий запрос и вызывал обработчик маршрутов, связанный с этим маршрутом (хранился в свойстве `RouteHandler` объекта `Route`) - в MVC приложении этим обработчиком был [MvcRouteHandler](https://msdn.microsoft.com/en-us/library/system.web.mvc.mvcroutehandler(v=vs.118).aspx), который брал на себя дальнейшую работу с запросом.

Маршруты в коллекцию `RouteTable.Routes` мы добавляли в процессе настройки приложения. Вот стандартный код настройки системы маршрутизации в MVC приложении:

	RouteTable.Routes.MapRoute(
	                name: "Default",
	                url: "{controller}/{action}/{id}",
	                defaults: new { controller = "Home", action = "Index", id = UrlParameter.Optional }
	            );

Где `MapRoute` - extension-метод, объявленный в пространстве имен `System.Web.Mvc`, который добавляет в коллекцию маршрутов в свойстве `Route` новый маршрут используя `MvcRouteHandler` в качестве обработчика.

Мы можем сделать это и самостоятельно:

    RouteTable.Routes.Add(new Route(
        url: "{controller}/{action}/{id}",
        defaults: new RouteValueDictionary(new { controller = "Home", action = "Index", id = UrlParameter.Optional }),
        routeHandler: new MvcRouteHandler())
        );

##Как организована система маршрутизации в ASP.NET 5##
RouteAsync
GetVirtualPath

##Настраиваем систему маршрутизации на использованием своего обработчика маршрутов##


##Как система маршрутизации настраивается для использования обработчика MVC##


##Настройка системы маршрутизации##
Использование MapRoute для MVC и не для MVC
Конвеншиал-базед подход
Аттрибут-базед подход

##Использование механизма опций на примере RouteOptions##

##Разбираемся с шаблоном маршрута##
Оптионал-параметр
Используем wildcard

##Использование областей в MVC 6##
area:exists

##Использование ограничителей маршрута##
RouteOptions.ConstraintMap
Создание своего ограничителя маршрутов
Использование своего ограничителя маршрутов в шаблоне

	services.Configure<RouteOptions>(options =>
	                                options
	                                .ConstraintMap
	                                .Add("test", typeof(TestRouteConstraint)));

##Best Effort Link Generation##
context.IsBounded - что это?

##Что было убрано из системы маршрутизации##
IgnoreRoute
RouteBase

##Какие возможности кастомизации мы имеем##
Наследуемся от TemplateRoute
Создаем свой ограничитель маршрутов