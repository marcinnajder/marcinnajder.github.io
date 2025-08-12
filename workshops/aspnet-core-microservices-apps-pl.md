---
layout: page
title: Tworzenie aplikacji Internetowych z wykorzystaniem ASP.NET Core
---
# Tworzenie aplikacji Internetowych z wykorzystaniem ASP.NET Core

Zarejestruj [tutaj](https://www.comarch.pl/szkolenia/programowanie/net-c/aplikacje-mikroserwisowe-w-aspnet-core/) lub napisz bezpośrednio
### Opis

Założeniem szkolenia jest poznanie przez uczestników głównych założeń architektury mikroserwisowej oraz frameworka ASP.NET Core wykorzystanego do jej implementacji. Uczestnicy zdobędą praktyczną wiedzą dotyczącą tworzenia własnych usług REST oraz gRPC. Dodatkowo dowiedzą się jak testować, a następnie wdrażać aplikacje mikroserwisowe z wykorzystaniem kontenerów Docker. Szkolenie prowadzone jest w formie wykładów, warsztatów i ćwiczeń praktycznych przy komputerach.

Czas trwania: 3 dni
### Szczegóły

- Wprowadzenie do .NET Core oraz ASP.NET Core
	- Główne cechy .NET Core, dotnet CLI
	- Ewolucja technologii webowych ASP.NET 
- Fundamenty ASP.NET Core
	- Dependency Injection
	- Konfiguracja aplikacji, logowanie (instrumentacja kodu), routing, startup
	- Middleware, pipeline aplikacji
	- Publikowanie oraz hosting aplikacji
- Web API
	- Routing, kontrolery, filtry
	- Obsługa rezultatu, formaty przesyłanych danych
	- Walidacja danych, model binding
	- Asynchroniczność po stronie serwera
- Minimal API
	- Routing, parameter binding, filtry
	- Obsługa zwracanego rezultatu, strumienie
- Swagger, OpenAPI
	- Definiowania metadanych w kodzie
	- Generowanie metadanych, dokumentacji, proxy
	- Narzędzia (Swashbuckle, NSwag)
- gRPC
	- Definicja wiadomości protobuf
	- Implementacja serwisu
	- Wywoływanie serwis
	- Logowanie, interceptory, obsługa błędów
	- Narzędzia (dotnet-grpc, grpcurl, grpcui)
- Docker
	- Tworzenie obrazów docker, multi-stage builds
	- Rejestr docker
	- Zarządzanie obrazami oraz kontenerami
	- docker-compose
- Testowanie aplikacji
	- Testy jednostkowe
	- Testy integracyjne
	- Narzędzia (postman, HttpRepl, pliki .http)
- Pozostałe zagadnienia dotyczące architektury mikroserwisowej
	- Zalety oraz wady architektury mikroserwisowej
	- Synchroniczna oraz asynchroniczna komunikacja pomiędzy serwisami
	- Wzorzec API Gateway
	- Tworzenie odpornych aplikacji (obsługa błędów, ponawiania, exponential backoff, circuit breaker, health checks, ...)
- .NET Aspire
