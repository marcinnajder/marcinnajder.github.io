---
layout: post
title: Running RIA Services on the top of Raven DB [C#]
date: 2012-05-06
tags:
  - csharp
series: asyncctp
---
In this post I will describe how to integrate [RIA Services](http://www.silverlight.net/learn/advanced-techniques/wcf-ria-services/get-started-with-wcf-ria-services) with document database called [Raven DB](http://ravendb.net/). If you are not familiar with those frameworks watch [this](http://mnajder.blogspot.com/2012/02/ria-services-and-raven-db-presentation.html) video and [this](http://mnajder.blogspot.com/2012/02/raven-db-presentation.html) one (my 2 presentations in polish). RIA Services integrates very easily with relational database by providing base implementations of domain services out of the box. Using the same pattern I have created the base implementation for services working with Raven DB. We will see later in this post what exactly the domain service is and how we can use it to build n-tier business application, but now let’s look at the Raven DB API. Let’s say we have the following data model:

```csharp
public partial class Person   
{          
    public string Id { get; set; }  
    public string Name { get; set; }  
      
    public string Biography { get; set; }  
    public string Photos { get; set; }  
    public double AverageRating { get; set; }  
}  
  
public partial class Movie  
{          
    public string Id { get; set; }  
  
    public string Name { get; set; }  
    public string Synopsis { get; set; }  
    public double AverageRating { get; set; }  
    public int? ReleaseYear { get; set; }  
  
    public string ImageUrl { get; set; }  
      
    public List<string> Genres { get; set; }  
    public List<string> Languages { get; set; }  
  
    public List<PersonReference> Cast { get; set; }  
    public List<PersonReference> Directors { get; set; }  
      
    public List<Award> Awards { get; set; }  
  
    public int CommentsCount { get; set; }  
    public string CommentsId { get; set; }  
}  
  
public class PersonReference  
{  
    public string PersonId { get; set; }  
    public string PersonName { get; set; }  
}  
  
public class Award  
{          
    public string Type  { get; set; }  
    public string Category { get; set; }  
    public int? Year { get; set; }  
    public bool Won { get; set; }  
  
    public string PersonId { get; set; }  
}
```


With this we can do some base CRUD operation:

```csharp
using (var store = new DocumentStore() { Url = "http://localhost:8080", }.Initialize())  
{                  
    using (var session = store.OpenSession())  
    {  
        var davidDuchowny = new Person() { Name = "David Duchovny" };  
        var demiMoore = new Person() { Name = "Demi Moore" };  
        session.Store(davidDuchowny);           // Id property is set here  
        session.Store(demiMoore);               // Id property is set here  
  
        var theJoneses = new Movie()  
        {  
            Name = "The Joneses",  
            Synopsis = "A seemingly perfect family moves into a suburban neighborhood, but when it comes to the " +  
                    "truth as to why they're living there, they don't exactly come clean with their neighbors.",  
            ReleaseYear = 2009,  
            AverageRating = 6.5,  
            Genres = new List<string>() { "Comedy", "Drama" },  
            Cast = new List<PersonReference>()  
        {  
            new PersonReference() { PersonId = davidDuchowny.Id, PersonName = davidDuchowny.Name},  
            new PersonReference() { PersonId = demiMoore.Id, PersonName = demiMoore.Name},  
        }  
        };  
        session.Store(theJoneses);              // Id property is set here  
  
        session.SaveChanges();                    
    }  
      
    using (var session = store.OpenSession())  
    {                      
        var movie = (from m in session.Query<Movie>()  
                     where m.Name == "The Joneses"  
                     select m).FirstOrDefault();  
        var people = session.Load<Person>(movie.Cast[0].PersonId, movie.Cast[1].PersonId);  
          
        session.Delete(movie);  
        session.Delete(people[0]);   
        session.Delete(people[1]);  
  
        session.SaveChanges();  
    }  
}
```

Raven DB stores data as a JSON documents:

![raven1](/assets/images/raven1.png)

Once we know how the Raven DB works we can look at the RIA Services. RIA Services gives us tools (Visual Studio extensions) and libraries that allow us to build n-tier business solution with the Silverlight or JavaScript application as a client and .Net Web Service as a server. We start building RIA Services solution from the domain service which is just a class deriving directly from DomainService or any other derived class depending on the type of a data store.
rave

![raven2](/assets/images/raven2.png)

If the relation database is a data store then we can use LinqToSqlDomainService, LinqToEntitesDomainService or DbDomainService (LINQ to Entities Code First approach). TableDomainService class allows us to store data inside Windows Azure Table. If we want to store data inside any other kind of data source we have to derive directly from the DomainService class. My base service called RavenDomainService derives from DomainService and it encapsulates the logic of initializing and saving the data inside Raven DB. Let’s look at the sample usage of this class:

```csharp
[EnableClientAccess]  
public class NetflixDomainService : RavenDomainService  
{  
    protected override IDocumentSession CreateSession()  
    {  
        var session = base.CreateSession();  
        session.Advanced.UseOptimisticConcurrency = true;  
        return session;  
    }  
  
    public IQueryable<Person> GetPeople()  
    {  
        return Session.Query<Person>();  
    }  
    public void InsertPerson(Person entity)  
    {  
        Session.Store(entity);  
    }  
    public void UpdatePerson(Person entity)  
    {  
        var original = ChangeSet.GetOriginal(entity);  
        Session.Store(entity, original.Etag.Value); // optimistic concurrency   
    }  
    public void DeletePerson(Person entity)  
    {  
        var original = ChangeSet.GetOriginal(entity);  
        Session.Store(entity, original != null ? original.Etag.Value : entity.Etag.Value);  // optimistic concurrency  
        Session.Delete(entity);  
    }  
  
    public IQueryable<Movie> GetMovies()  
    {  
        return Session.Query<Movie>();  
    }  
    public void InsertMovie(Movie entity)  
    {  
        Session.Store(entity);  
    }  
    public void UpdateMovie(Movie entity)  
    {  
        var original = ChangeSet.GetOriginal(entity);  
        Session.Store(entity, original.Etag.Value); // optimistic concurrency   
    }  
    public void DeleteMovie(Movie entity)  
    {  
        var original = ChangeSet.GetOriginal(entity);  
        Session.Store(entity, original != null ? original.Etag.Value : entity.Etag.Value);  // optimistic concurrency  
        Session.Delete(entity);  
    }  
}
```

After adding “WCF RIA Services link” from the Silverlight project to projects containing domain service implemented above and rebuilding the whole solution, Visual Studio will generate appropriate code on the client side and we are ready to write code very similar to that presented earlier:

```csharp
async public void BlogPost()  
{  
    var context = new NetflixDomainContext();  
      
    var davidDuchowny = new Person() { Name = "David Duchovny" };  
    var demiMoore = new Person() { Name = "Demi Moore" };  
    context.Persons.Add(davidDuchowny);  
    context.Persons.Add(demiMoore);  
  
    await context.SubmitChanges().AsTask();         // Id property is set here  
  
    var theJoneses = new Movie()  
    {  
        Name = "The Joneses",  
        Synopsis = "A seemingly perfect family moves into a suburban neighborhood, but when it comes to the " +  
                "truth as to why they're living there, they don't exactly come clean with their neighbors.",  
        ReleaseYear = 2009,  
        AverageRating = 6.5,  
        Genres = new List<string>() { "Comedy", "Drama" },  
        Cast = new List<PersonReference>()  
        {  
            new PersonReference() { PersonId = davidDuchowny.Id, PersonName = davidDuchowny.Name},  
            new PersonReference() { PersonId = demiMoore.Id, PersonName = demiMoore.Name},  
        }  
    };  
    context.Movies.Add(theJoneses);  
  
    await context.SubmitChanges().AsTask();         // Id property is set here  
  
  
    var context2 = new NetflixDomainContext();  
      
    var movie = (await context2.Load  
        (  
            from m in context2.GetMoviesQuery()  
            where m.Name == "The Joneses"  
            select m  
        )).FirstOrDefault();  
  
    var people = (await context2.Load  
        (  
            from p in context2.GetPeopleQuery()  
            where p.Id == movie.Cast[0].PersonId || p.Id == movie.Cast[1].PersonId  
            select p  
        )).ToArray();  
  
    context2.Movies.Remove(movie);  
    context2.Persons.Remove(people[0]);  
    context2.Persons.Remove(people[1]);  
    await context2.SubmitChanges().AsTask();  
}
```

RIA Services gives us a very nice implementation of [Unit Of Work](http://martinfowler.com/eaaCatalog/unitOfWork.html) and [Identity Map](http://martinfowler.com/eaaCatalog/identityMap.html) patterns so the code using DAL frameworks like LINQ to SQL, LINQ to Entities, NHibernate or Raven DB is very similar to that written on the client side. There is one thing worth mentioning at this point, RIA Services needs to know which property identifies the entity object. I didn’t show you how to do it yet but it will be presented in a moment, I am writing about it here because without this little hint, the code above wouldn’t even compile.

The last thing I would like to explain is how the [optimistic concurrency](http://martinfowler.com/eaaCatalog/optimisticOfflineLock.html) pattern has been implemented in case of Raven DB. Raven DB has metadata mechanism which means that each JSON document besides the actual data has additional information like document type, last modification time, etag and so on. Etag is a single value of type Guid and its goal is very similar to Timestamp or Row Version column data type in SQL Server. Raven DB server changes the value of Etag internally every time the document is changing. With this Raven DB client API allows us to enable optimistic concurrency checking  on the session object level via UseOptimisticConcurrency property. There is one problem when it comes to integration with RIA Services optimistic concurrency mechanism. Etag is part of the metadata instead of document itself and it is handled by the session object internally. RIA Services doesn’t have any notion of metadata information, all data is stored inside the entity objects so we need to add additional property storing etag value which will be used on the RIA Services level but on the Raven DB level this value will be mapped into etag metadata field.

```csharp
public interface IEtag  
{  
    Guid? Etag { get; set; }  
}  
  
[MetadataTypeAttribute(typeof(PersonMetadata))]  
public partial class Person : IEtag  
{  
    [JsonIgnore] // do not store value of this property  
    public Guid? Etag { get; set; }  
  
    public class PersonMetadata  
    {  
        [Key]  
        public object Id { get; set; }  
  
        [RoundtripOriginal] // send back value of this property to client  
        public object Etag { get; set; }  
    }          
}  
  
[MetadataTypeAttribute(typeof(MovieMetadata))]  
public partial class Movie : IEtag  
{  
    [JsonIgnore] // do not store value of this property  
    public Guid? Etag { get; set; }  
  
    public class MovieMetadata  
    {  
        [Key]  
        public object Id { get; set; }  
  
        [Display(Name="nazwa")]  
        [Required]  
        [StringLength(200)]              
        public object Name { get; set; }  
  
        [Range(1900, 2012)]  
        public object ReleaseYear { get; set; }  
  
        [RoundtripOriginal] // send back value of this property to client  
        public object Etag { get; set; }  
    }  
}
```

Attributes like Key, Display, Required, Range, RoundtripOriginal and MetadataType are a part of the standard .net mechanism called data annotations. RIA Services similar to other frameworks like ASP.MVC or Dynamic Data uses them to define validation rules or UI controls in layout in declarative way. Key attributes tell RIA Services which property identifies the entity object, RoundtripOriginal attributes tell that the value of the property is changed on the server side so the new value should be sent back to the client after calling server method. JsonIgnore is specific to Raven DB and it means that this property should be ignore during serialization and deserialization process. IEtag interface was introduced as a marker interface so all entities implementing this interface are treated specially by listener code. Listeners are a part of Raven DB client API and we can think of them as of a client side triggers executed in various situations.For instance during JSON document conversion process (IDocumentConversionListener) or before and after document is stored inside Raven DB (IDocumentStoreListener).

```csharp
public class Global : System.Web.HttpApplication  
{  
    private static IDocumentStore _documentStore;  
  
    protected void Application_Start(object sender, EventArgs e)  
    {  
        var etagSetterListener = new EtagSetterListener();  
  
        _documentStore = new DocumentStore() { Url = "http://localhost:8080",  }  
            .RegisterListener(etagSetterListener as IDocumentConversionListener)  
            .RegisterListener(etagSetterListener as IDocumentStoreListener)  
            .Initialize();  
          
        DomainService.Factory = new RavenDomainServiceFactory(_documentStore, DomainService.Factory);  
    }  
}  
  
public class EtagSetterListener : IDocumentConversionListener, IDocumentStoreListener  
{  
    #region IDocumentConversionListener  
  
    public void EntityToDocument(object entity, RavenJObject document, RavenJObject metadata)  
    {  
    }  
  
    public void DocumentToEntity(object entity, RavenJObject document, RavenJObject metadata)  
    {  
        var etag = entity as IEtag;  
        if (etag != null)  
        {  
            etag.Etag = Guid.Parse(metadata["@etag"].ToString());  
        }  
    }   
    #endregion  
  
  
    #region IDocumentStoreListener  
      
    public bool BeforeStore(string key, object entityInstance, RavenJObject metadata)  
    {  
        return false;  
    }  
  
    public void AfterStore(string key, object entityInstance, RavenJObject metadata)  
    {  
        var etag = entityInstance as IEtag;  
        if (etag != null)  
        {  
            etag.Etag = Guid.Parse(metadata["@etag"].ToString());  
        }  
    }  
  
    #endregion  
}  
  
  
public class RavenDomainServiceFactory : IDomainServiceFactory  
{  
    private readonly IDocumentStore _documentStore;  
    private readonly IDomainServiceFactory _domainServiceFactory;  
  
    public RavenDomainServiceFactory(IDocumentStore documentStore, IDomainServiceFactory domainServiceFactory)  
    {  
        _documentStore = documentStore;  
        _domainServiceFactory = domainServiceFactory;  
    }  
  
    public DomainService CreateDomainService(Type domainServiceType, DomainServiceContext context)  
    {  
        var service = _domainServiceFactory.CreateDomainService(domainServiceType, context);  
  
        if (service is RavenDomainService)  
        {  
            ((RavenDomainService)service).DoumentStore = _documentStore;      
        }  
  
        return service;  
    }  
  
    public void ReleaseDomainService(DomainService domainService)  
    {  
        _domainServiceFactory.ReleaseDomainService(domainService);  
    }  
}
```

IDomainServiceFactory interface allows us extend the process of creation and destruction of domain services instance on the server side. Finally let’s see how the RavenDomainService class has been implemented:

```csharp
public abstract class RavenDomainService : DomainService  
{  
    public IDocumentStore DoumentStore { get; set; }  
  
    private IDocumentSession _session;  
    protected IDocumentSession Session  
    {  
        get  
        {  
            if (_session == null && DoumentStore != null)  
            {  
                _session = CreateSession();  
            }                      
            return _session;  
        }  
    }  
  
    private IDocumentSession _refreshSession;  
    private IDocumentSession RefreshSession  
    {  
        get  
        {  
            if (_refreshSession == null && DoumentStore != null)  
            {  
                _refreshSession = CreateSession();  
            }  
            return _refreshSession;  
        }  
    }  
  
    protected virtual IDocumentSession CreateSession()  
    {  
        var session = DoumentStore.OpenSession();  
        return session;  
    }  
  
    protected override void Dispose(bool disposing)  
    {  
        if (disposing)  
        {  
            if (_session != null)  
            {  
                _session.Dispose();  
                _session = null;  
            }  
            if (_refreshSession!= null)  
            {  
                _refreshSession.Dispose();  
                _refreshSession = null;  
            }       
        }  
        base.Dispose(disposing);  
    }  
  
    protected override int Count<T>(IQueryable<T> query)  
    {  
        return query.Count();  
    }  
  
    protected override bool PersistChangeSet()  
    {  
        try  
        {  
            this.Session.SaveChanges();  
        }  
        catch (ConcurrencyException exception)  
        {  
            var items =  base.ChangeSet.ChangeSetEntries  
                .Where(changeSet =>  
                           {  
                               var etag = changeSet.Entity as IEtag;  
                               if (etag == null)  
                                   return false;  
                               return etag.Etag == exception.ExpectedETag;  
                           })  
                .ToArray();  
              
            foreach (var item in items)  
            {  
                var o = ChangeSet.GetChangeOperation(item.Entity);  
                if(o == ChangeOperation.Delete)  
                {  
                    item.IsDeleteConflict = true;  
                }  
                else  
                {  
                    item.ConflictMembers = new[] { "unknown" };  
                    var id = Session.Advanced.GetDocumentId(item.Entity);  
                    item.StoreEntity = RefreshSession.Load<object>(id);  
                }  
            }  
                   
            this.OnError(new DomainServiceErrorInfo(exception));  
  
            if (!base.ChangeSet.HasError)  
            {  
                throw;  
            }  
  
            return false;  
        }  
  
        return true;  
    }  
}
```

I know, I know, I hear you saying: "What did you do all this for ??? Raven DB gives you a wonderful client API for Silverlight". I totally agree with you. I am not trying to say that you should use RIA Services instead of Raven DB directly via its Silverlight API. Those two frameworks are slightly different things. They have a different functionalities, they are designed to resolve a bit different kinds of problems. Raven DB is a database system so it is focused mainly on data storage. RIA Services solves many common problems for a business n-tier application like server and client side validation, easy integration with DataGrid or DataForm UI controls, sharing common code between client and server, marshaling exceptions and so on. For me, those two frameworks can work great together and such cooperation seems to be very reasonable. I was searching google for any samples showing how to integrate them but I couldn’t find it. So I did my own !:) I hope you find it useful.

[Download](http://archive.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=mnajder&DownloadId=15895)

