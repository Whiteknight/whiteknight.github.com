---
layout: post
categories: [Architecture]
title: Thin Controllers, Thin View Models
---

Everything on the web is MVC now (or MVP, or MVVM, which are separated more by nuance and discipline than by actual structure) and for good reason: It's a natural and straight-forward way to separate the concerns of display, data and logic. I find that getting trapped in this MVC structure, and trying to use it for more than it is intended for, can lead to big trouble.

## The MVC Trap

When you look at something like ASP.NET MVC, when you create a new project a few folders are already created for you: `/Models`, `/Views` and `/Controllers`. With the Rails-inspired naming conventions enabled by default, it's very easy to create a `FooController` whose method names correspond to file names in the `/Views/Foo/` folder. Various tools run with this. You can often click or right-click on a controller method and automatically be taken to the corresponding view file, with the file being created for you on the fly if needed. This is easy, so easy that many developers get lured into a trap.

When you have two roads in front of you, and you're not quite sure where they lead to off in the distance, it's very easy to decide to pick which looks easier at the start. When the MVC system and the associated tools are creating methods for you on your controller, and your controller is attached by convention to automatically-created views, the easy road tells us that this is our structure and this is where our logic goes. So we start writing code, putting display logic into our View and putting some data-munging logic into our Model, and dumping most of the rest of the logic into our Controller. The end result, those goal posts off in the distance we didn't quite see at first, is that our controllers are fat, our views are polluted, and our models are bloated with unrelated business logic. Then, when somebody says we need to make an alternate view, because we need to display much of the same logic in a different format for a specific subset of clients, everything goes to hell. We curse our tools, stomp our feet, and search the googles for the next shiny thing that promises to make all our cares go away.

When you look at Microsoft tech demos, especially those for new ASP.NET MVC releases, you see people wanting to reuse data models to prevent code duplication. You create a POCO class and attach it to the EntityFramework DbContext, then you use the Visual Studio templating tools to create a controller for your entity class with automatically-generated views for the CRUD operations. Everything works end-to-end, with we the programmers needing to write very little code compared to how much is generated for us, and we call ourselves geniuses for leveraging code generation and avoiding the pitfalls of the layered architectures of our forefathers. Why did we have those things in the first place? Who needs layers? But now our models are serving many masters. We start fleshing our entity classes out into a full Domain Model with methods for business logic, but we need some validation methods down in the data code and we need some formatting methods to help with displaying data in the View. Now we have a bunch of methods with warnings on them like "Don't call this method without an active DbContext!" or "Don't call this method until the user has been validated!". Then, when the business guys want us to associate multiple colors with every product instead of just one, and the storage format needs to change to use another table with a foreign key, now we have to rewrite our entire view because the same model that represents the database also represents the Domain Model and the View Model. Far from making our lives simpler, we are now living in hell where every little change in one domain forces massive rewrites in the others.

Then the networking guys are telling us that the Shipping Cost calculations are too expensive, and we want to break that logic out into a separate service so we can offload the calculations from the webserver and distribute them onto some helper servers outside the DMZ. How do we possibly accomplish that? All our logic is in our controllers (and our Views, and our entity models) and it's impossible to tease it all out without just rewriting the whole damned thing.

When Stefan Tilkov tells us [Don't start with a monolith](http://martinfowler.com/articles/dont-start-monolith.html), this is what he is talking about. If you go down this path, your mess will never be detangled and you'll end up throwing everything out when the requirements change too much.

## MVC: Three Separate, Single Responsibilities

The solution to a lot of these problems is kind of simple, if you keep it in mind from the beginning: SRP. **The Single Responsibility Principle** tells us that [classes should do one thing and serve one master](/2015/03/14/srp.html). So, what are the single responsibilities of each of the MVC components?

* **Model**: To hold data needed for the view, in a format that the View requires. The Model is a servant of the View, and only changes when the View needs it to change.
* **View**: To allow the user to view and interact with the data. This only changes when the needs of the user changes.
* **Controller**: To mediate between the user domain and the business domain. This only changes when the API requirements of the Application Service Layer changes.

Boom. Three parts, each with a single purpose and a single master. But this almost raises more questions than it answers: Where does all that logic go, which used to live in our fat controller methods, and in our polluted View, and in our bloated Model class?

### ViewModel Responsibilities

First and foremost, if our model is a proper "View Model" and only serves the View, that means we're going to need to have another class which represents the database storage format and yes, we're probably also going to want another more-or-less duplicate to serve as our domain model. We have three classes here because the data serves three masters, and lives in three different realms and needs to satisfy three different sets of requirements. You're going to need to do some mapping, but a conventions-based solution like AutoMapper will get you 90% of the way there with minimal effort. Maybe AutoMapper isn't what you use long-term, but it can be a good start nonetheless. If you isolate the automapper dependency and abstract it behind some kind of `IMapper<TFrom, TTo>` interface, you can always go back and fix it later with minimal effort.

Your three models are:

* **Your Data Model** which serves the needs of our data storage mechanism. Properties on these classes probably have a one-to-one correspondence with columns in your database, and methods on these classes have to do with low-level validation and translating data storage formats into something usable by the program.
* **Your Domain Model** which holds your business logic. The Domain Model is set up to meet the needs of your business, divorced from any concerns of data storage or data display.
* **Your ViewModel** which serves the needs of your UI. The ViewModel holds information needed by the View, and provides data manipulation and front-line validation methods required by the View.

### View Responsibilities

The View now only deals with display concerns, and we don't need to worry that it's getting its fingers down into the business logic or the data storage logic. All the logic bloat which used to live in your View now probably lives in your ViewModel, because the ViewModel is the data-holding servant of the View.

The View is conceptually very simple because it is only concerned with presentation. Holding and organizing data to be displayed is the role of the ViewModel. In this sense the View is very dumb: Take data out of the ViewModel. Show it as-is to the user. Repeat with the next piece of data. When we talk about Views being dumb, we are obviously only talking about it from the point-of-view of the server. From the client perspective, JavaScript in the View might have just as much complicated logic, if not more, than the entire rest of your application combined.

### Controller Responsibilities

The Controller is now a simple mediator. It takes requests from the user, translates those into domain requests, receives back a domain response, and translates that into a response for the user. Any other logic which used to live in your Controller now lives in your Domain Model or in an Application Service Layer of some sort. Not only is it easier to do things like unit test the controller in this setup, but if the implementation is thin enough and if the Service Layer is tested already, you might not need to bother testing your Controller at all! Here's a basic MVC controller method which illustrates what a thin controller method may look like:

{% highlight csharp %}
public ActionResult GetTheData(UserRequestModel userRequest)
{
  m_requestValidator.Validate(userRequest);
  DomainRequest domainRequest = m_requestTranslator.Translate(userRequest);
  DomainResponse domainResponse = LocationApplicationService<DataService>().Handle(domainRequest);
  UserResponseModel userResponse = m_requestTranslator.Translate(domainResponse);
  return View(userResponse);
}
{% endhightlight %}

#### How Low Can You Go?

It's worth pointing out that we could probably make this controller method even smaller by recognizing that we have a repeatable pattern, and putting in some effort to abstract our translator behind a generic interface:

{% highlight csharp %}
public ActionResult GetTheData(UserRequestModel userRequest)
{
  UserResponseModel userResponse = ControllerHelper.Dispatch<UserRequestModel, DomainRequest, DomainResponse, UserResponseModel, DataService>(userRequest);
  return View(userResponse);
}
{% endhightlight %}

But then again, many people (and many code analysis tools) would likely be pretty unhappy about all those type parameters and by the extremely abstract nature of this `Dispatch` method, not to mention the implied reliance on a Service Locator to find our `IRequstTranslator<UserRequestModel, DomainRequest>`, `IRequestTranslator<DomainResponse, UserResponseModel>`, our validator, our service, and anything else we wanted in there. Controller methods should be thin, but there is some price to pay if you make it *too thin*.

In either case, this is all the logic that should appear in your controller method. Validate the request. Translate the request. Dispatch the request. Receive the response, translate the response, return the response.

#### WebAPI Methods

The implementation of a webservice using WebAPI is identical, just returning the translated response DTO instead of returning a View:

{% highlight csharp %}
public UserResponseDto GetTheData(UserRequestDto userRequest)
{
  m_requestValidator.Validate(userRequest);
  DomainRequest domainRequest = m_requestTranslator.Translate(userRequest);
  DomainResponse domainResponse = LocationApplicationService<DataService>().Handle(domainRequest);
  UserResponseDto userResponse = m_requestTranslator.Translate(domainResponse);
  return userResponse;
}
{% endhightlight %}

## What's The Payoff?

After doing all this work, making multiple models for the different layers of your application, moving logic out of the View into the ViewModel, and making our Controllers thin, what have we acheived? What has all our effort bought us?

1. MVC Controllers are tied to the View and HTTP and the web. Pulling logic out of there makes your logic not depend on any of those things. This means all your application logic exists separately from the web, which means it can be used separately without any issue.
2. Logic is easier to test, because now we can run fast, cheap *unit tests* on our Application Service and Domain Model instead of having to set up expensive *integration tests* on our controllers and views.
3. The logic of the application service is now reusable in different formats. We can create new APIs, or new versioned APIs, or new front-ends with the same logic without any issues.
4. We open our controllers up to Dependency Injection, which will allow us to substitute different implementations based on outside factors. This makes things like A/B testing, or custom per-client behaviors in a multi-tenancy system more doable. (The dependency injector can peek at our HTTP headers and authentication tokens, and select service instances depending on what it finds there, etc).

Then we can move the `DataService` into a service layer in a separate assembly, and reuse that logic elsewhere when the boss asks us for a separate view for some clients, or a report that feeds off the same data, or anything like that.

## The Road To Microservices, Maybe

You'll notice that a system like this has a vertical separation of concerns instead of the horizontal separation into layers of earlier systems. The Controller calls the Application Service, which interacts with the Domain Model and eventually creates requests into the database. Separating a system like this out into microservices is relatively easy because we already have problem-domain separation and we already have an API suitable for remote calls. That's if Microservices are worth the hassle for your team, which they won't necessarily be. It's not one size to fit all, but it is an available option for people who find they do need it.

When Martin Fowler says that we should [Start with a monolith first](http://martinfowler.com/bliki/MonolithFirst.html) and break out into microservices later as needed, he's expecting us to already have a nice clean architecture like this. If you don't have that, if you don't have the discipline to properly structure your monolith with an eye towards proper structure and eventual decomposition, this isn't the road for you. Either you need to decide that Microservices are not your final destination (a very difficult, nerve-wracking decision to make so early!) or you need to decide to go with Microservies from the beginning. You're paying the Microservice premium too early, or you'll be paying the massive cleanup and refactoring costs too late. It's a tough decision if you're in this boat, but one that needs to be made.
