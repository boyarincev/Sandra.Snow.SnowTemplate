---
layout: post
title: Asp .Net для Asp .Net MVC разработчиков: Часть 1 Глобальный объект Asp .Net приложения
categories: Asp-Net
published: draft
---

**Информация в статье актуальна для Asp .Net 4.6**

Платформа Asp .Net изначально была построена для использования с Web Forms приложениями, а поддержка Asp .Net MVC была добавлена в нее позже. Сама по себе платформа самодостаточна и имеет множество полезных возможностей, но большинство современных разработчиков Asp .Net MVC не имеют опыта разработки на Web Forms и поэтому не знают о этих возможностях.

В этой серии статей я расскажу о строении платформы и возможностях, которые она предоставляет. Основа для статей взята из книги Адама Фримена [Pro Asp Net MVC Platform](http://www.apress.com/9781430265412)

##Глобальный объект приложения##
Глобальным объектом приложения является файл Global.asax, который находится в корне вашего веб-приложения. Этот файл используется во всех типах asp .net приложений (Web Forms, MVC, WebApi и т.д.).

Файл физически состоит из двух файлов:
- Global.asax - разметка
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



