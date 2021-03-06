# ViewBag, ViewData & TempData

The **"Music Store"** web application does use only the **"ViewBag"** so we will write some tests for it. The **"ViewData**" and the **"TempData"** use similar syntax.

## Testing with ViewBag entry

Let's test something simple - the HTTP Get overload of the **"Login"** action in the **"AccountController"**:

```c#
public IActionResult Login(string returnUrl = null)
{
	ViewBag.ReturnUrl = returnUrl;
	return View();
}
```

We need a new dependency (again?!) - **"MyTested.AspNetCore.Mvc.ViewData"**:

```json
"dependencies": {
  "dotnet-test-xunit": "2.2.0-*",
  "xunit": "2.2.0-*",
  "Moq": "4.6.38-*",
  "MyTested.AspNetCore.Mvc.Authentication": "1.0.0",
  "MyTested.AspNetCore.Mvc.Caching": "1.0.0",
  "MyTested.AspNetCore.Mvc.Controllers": "1.0.0",
  "MyTested.AspNetCore.Mvc.DependencyInjection": "1.0.0",
  "MyTested.AspNetCore.Mvc.EntityFrameworkCore": "1.0.0",
  "MyTested.AspNetCore.Mvc.Http": "1.0.0",
  "MyTested.AspNetCore.Mvc.ModelState": "1.0.0",
  "MyTested.AspNetCore.Mvc.Models": "1.0.0",
  "MyTested.AspNetCore.Mvc.Options": "1.0.0",
  "MyTested.AspNetCore.Mvc.Session": "1.0.0",
  "MyTested.AspNetCore.Mvc.ViewActionResults": "1.0.0",
  "MyTested.AspNetCore.Mvc.ViewData": "1.0.0", // <---
  "MusicStore": "*"
},
```

We need to add the **"ViewData"** features, because the **"ViewBag"** is actually a [dynamic version of it](https://github.com/aspnet/Mvc/blob/dev/src/Microsoft.AspNetCore.Mvc.ViewFeatures/Controller.cs#L91).

I hope you remember how we tested session and cache. Well, the **"ViewBag"** (**"ViewData"** and **"TempData"** too) is no different:

```c#
[Fact]
public void LoginShouldHaveReturnUrlInTheViewBag()
{
    var returnUrl = "Test/Return/Url";

    MyController<AccountController>
        .Instance()
        .Calling(c => c.Login(returnUrl))
        .ShouldHave()
        .ViewBag(viewBag => viewBag // <---
            .ContainingEntry("ReturnUrl", returnUrl))
        .AndAlso()
        .ShouldReturn()
        .View();
}
```

## Testing with multiple ViewBag entries

Let's write another one - for the HTTP Get overload of the **"Create"** action in **"StoreManagerController"**:

```c#
public IActionResult Create()
{
	ViewBag.GenreId = new SelectList(DbContext.Genres, "GenreId", "Name");
	ViewBag.ArtistId = new SelectList(DbContext.Artists, "ArtistId", "Name");
	return View();
}
```

And our test:

```c#
[Fact]
public void CreateShouldHaveValidEntriesInViewBag()
{
    var genres = new[]
    {
        new Genre { GenreId = 1, Name = "Rock" },
        new Genre { GenreId = 2, Name = "Rap" }
    };

    var artists = new[]
    {
        new Artist { ArtistId = 1, Name = "Tupac" },
        new Artist { ArtistId = 2, Name = "Biggie" }
    };

    MyController<StoreManagerController>
        .Instance()
        .WithDbContext(db => db
            .WithEntities(entities =>
            {
                entities.AddRange(genres);
                entities.AddRange(artists);
            }))
        .Calling(c => c.Create())
        .ShouldHave()
        .ViewBag(viewBag => viewBag // <---
            .ContainingEntries(new
            {
                GenreId = new SelectList(
					From.Services<MusicStoreContext>().Genres, "GenreId", "Name"),
					
                ArtistId = new SelectList(
					From.Services<MusicStoreContext>().Artists, "ArtistId", "Name")
            }))
        .AndAlso()
        .ShouldReturn()
        .View();
}
```

The **"ContainingEntries"** call is equivalent to this one:

```c#
.ContainingEntries(new Dictionary<string, object>
{
	["GenreId"] = new SelectList(
		From.Services<MusicStoreContext>().Genres, "GenreId", "Name"),
		
	["ArtistId"] = new SelectList(
		From.Services<MusicStoreContext>().Artists, "ArtistId", "Name")
}))
``` 

Both methods will validate whether the total number of entries in the **"ViewBag"** is equal to the total number you provide in the test. For a sanity check - remove the **"ArtistId"** property from anonymous object and run the test again:

```text
When calling Create action in StoreManagerController expected view bag to have 1 entry, but in fact found 2.
```

If you do not want the total number of entries validation, just use:

```c#
.ViewBag(viewBag => viewBag // <---
	.ContainingEntry("GenreId", new SelectList(
		From.Services<MusicStoreContext>().Genres, "GenreId", "Name"))
	.ContainingEntry("ArtistId", new SelectList(
		From.Services<MusicStoreContext>().Artists, "ArtistId", "Name")))
```

## ViewData and TempData

**"ViewData"** have the same API:

```c#
MyController<SomeController>
	.Instance()
	.Calling(c => c.SomeAction())
	.ShouldHave()
	.ViewData(viewData => viewData // <---
		.ContainingEntry("SomeKey", someValue))
	.AndAlso()
	.ShouldReturn()
	.View();
```

**"TempData"** too, but you will need the **"MyTested.AspNetCore.Mvc.TempData"** package:

```c#
MyController<SomeController>
	.Instance()
	.Calling(c => c.SomeAction())
	.ShouldHave()
	.TempData(tempData => tempData // <---
		.ContainingEntry("SomeKey", someValue))
	.AndAlso()
	.ShouldReturn()
	.View();
```

Additionally, you can populate the **"TempData"** dictionary before the actual action call:

```c#
MyController<SomeController>
	.Instance()
	.WithTempData(tempData => tempData
		.WithEntry("SomeKey", someValue))
	.Calling(c => c.SomeAction())
	.ShouldReturn()
	.View();
```

## Section summary

We saw how easy it is to test with **"ViewBag"**, **"ViewData"** and **"TempData"**. Their fluent assertion APIs are very similar to the **"Session"** and **"Cache"** ones. But enough about controllers, let's finally move on to [View Components](/tutorial/viewcomponents.html)!