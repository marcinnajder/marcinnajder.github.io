---
layout: page
title: Tworzenie aplikacji Internetowych z wykorzystaniem ASP.NET Core
---
# Tworzenie aplikacji Internetowych z wykorzystaniem ASP.NET Core

Zarejestruj [tutaj](https://www.comarch.pl/szkolenia/programowanie/net-c/tworzenie-aplikacji-internetowych-z-wykorzystaniem-aspnet-core/) lub napisz bezpośrednio
### Opis

Założeniem szkolenia jest poznanie przez uczestników wieloplatformowego frameworka ASP.NET Core. Oprócz teoretycznej wiedzy uczestnicy nabędą umiejętności praktycznych umożliwiających budowę uporządkowanej, zwartej, logicznej aplikacji z wykorzystaniem wzorca projektowego MVC, który wymusza podział aplikacji na trzy niezależne warstwy: model danych, interfejs graficzny oraz logikę działania. Protokół HTTP nie służy wyłącznie do zwracanie stron www, dlatego uczestnicy zdobędą również umiejętności projektowania oraz budowy Web API w architekturze REST. 

Czas trwania: 3 dni
### Szczegóły

- Wprowadzenie .NET Core
	- Czym jest .NET Standard 2.0 (wieloplatformowość, architektura)
	- Narzędzia (msbuild, cmd, nuget, docker)
	- Krótka historia ASP.NET
- Entity Framework Core
	- Źródła danych stosowane w ASP.NET Core
	- Opisywanie modelu za pomocą encji POCO
	- CRUD - Tworzenie relacyjnej bazy danych z modelu, pobieranie oraz modyfikacja danych z wykorzystaniem Entity Framework Code First
- ASP.NET Core - Podstawy
	- Hosting (kestrel, konfiguracja)
	- Dependency Injection,
	- Wzorzec Repository
	- Middleware
	- Omówienie wbudowanych Middlewareów (logowanie, obsługa błędów, CORS, serwowanie plików statycznych)
	- Zasady działania mechanizmów routing'u
	- Obszary stosowania mechanizmów routing'u (Areas)
	- Przetwarzanie żądania HTTP
	- Budowa żądania HTTP
	- Architektura REST
	- Filtry (opis istniejących, tworzenie własnych)
- ASP.NET Core - MVC
	- Architektura MVC
	- Podstawowe mechanizmy służące do budowy kontrolerów w architekturze MVC
	- Klasa ActionResult i jej zastosowanie w kontrolerach
	- Asynchroniczne operacje kontrolera z wykorzystaniem typów Task
	- Wykorzystanie klas ViewData oraz TempData, w celu usprawnienia kontrolerów
	- Widok
		- Sposoby definiowania widoków
		- Definiowanie układu strony
		- Składnia Razor
		- Typowane i nietypowane widoki
		- Metody pomocnicze HTML (tworzenie własnych)
		- Szablony (tworzenie własnych)
		- Sekcje
		- Mechanizmy partiaviews oraz child actions
- ASP.NET Core - Zaawansowany
	- Testowanie aplikacji
		- Projektowanie aplikacji pod kątem procesu testowania
		- Proces integracji testów jednostkowych z aplikacją
		- Przydatność wzorców Dependency Injection oraz Repository podczas testowania aplikacji
	- Bezpieczeństwo
		- Autentykacja
		- Autoryzacja
		- Zabezpieczenia przeciwko XSS oraz CSRF
	- Hosting Kestrel/IIS
		- Metody wdrażania aplikacji
		- Konfiguracja
		- dotnet publish-iis
	- Mechanizmy Cache'owania
	- Client-Side Development, Node Services