---
layout: single
title:  "Secret Santa with MVC"
date:   2021-10-04 18:40:03 -0600
categories: mvc asp.net c# web-dev
---


MVC, short for Model View Controller, is an architectural design pattern that separates an application into three components. The first is the *model* component, which encapsulates all logic related to handling application data. The second component, the *view* holds all UI logic for the application, and the final component, the *model* contains all application business logic and acts as an interface between the view and model.
MVC architecture can provide many benefits, such as a loose coupling of components, easier testability, code reuse, and parallel development.

## Demonstration: A Secret Santa Generator

Recently, I found myself in need of a way to assign secret Santa's amongst my friends without being able to gather in person (I bet you can guess why). I found there are many websites out there just for this purpose, but they all had a problem: I didn't make them. I wanted to both do a secret Santa this year *and* impress my non-programmer friends, so I decided to create a website with MVC architecture.

### The Basic Workflow

For my site, users are initially presented with two options: create a room or join a room. If users create a room, they're prompted to create a unique room code that they can share with their friends. After creating a room, the user is then asked to input a name, after which they're automatically taken to the room where they can wait for their friends to join. Once they're ready, everyone in the room can click a link to be assigned a gift recipient.

### The Platform

I created my project using ASP.NET's MVC framework in C#. If you're curious as to how to set this kind of project up Microsoft has good documentation on the matter: [ASP.NET MVC Setup](https://docs.microsoft.com/en-us/aspnet/core/tutorials/first-mvc-app/start-mvc?view=aspnetcore-5.0&tabs=visual-studio)

### Creating Rooms

We want a way to store many rooms and access them via room codes. A dictionary is the perfect data structure for the job. To make a dictionary with room code and room key/pair values accessible to our home controller, we can register an instance of the ConcurrentDictionary class as a singleton inside of our startup.cs file:

{% highlight csharp%}
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllersWithViews();

    services.AddSession(options => {
        options.IdleTimeout = TimeSpan.FromDays(1);
        options.Cookie.HttpOnly = true;
        options.Cookie.IsEssential = true;
    });

    services.AddSingleton<IDictionary<string, SecretSantaRoom>>(new ConcurrentDictionary<string, SecretSantaRoom());
}
{% endhighlight %}

The dictionary we registered is now available to our controller via constructor injection.

For a user to create a room, they just have to submit a room code that is unique. If the room code they entered isn't already in use, we add a new Room instance to our dictionary with the room code as the key and render the EnterName view. If the user didn't enter a room code or entered a room code already in use, then we re-render the view with an error message.

{% highlight csharp%}
public class HomeController : Controller
{
    private readonly ILogger<HomeController> _logger;

    public IDictionary<string, SecretSantaRoom> Rooms { get; }

    public HomeController(IDictionary<string, SecretSantaRoom> rooms)
    {
        Rooms = rooms;
    }

    [HttpGet]
    public IActionResult CreateRoom()
    {
        return View();
    }

    [HttpPost]
    public IActionResult CreateRoom(CreateRoomViewModel vm)
    {
        if(!ModelState.IsValid)
        {
            return View();
        }
        else
        {
            var newRoom = new SecretSantaRoom();

            if (Rooms.TryAdd(vm.RoomCode, newRoom))
            {
                return View(nameof(EnterName), new EnterNameViewModel { RoomCode = vm.RoomCode });
            }
            else {
                ModelState.AddModelError("RoomExists", "Room already exists");
                return View();
            }

        }
    }
}
{% endhighlight %}

{% highlight html%}
@model CreateRoomViewModel

<head>
    <link rel="stylesheet" href="~/css/InputError.css" />
    <link rel="stylesheet" href="~/css/christmas.css" />
</head>
<body>
    <h1>Create Session</h1>
    <form asp-action="CreateRoom">
        <div asp-validation-summary="All"></div>
        <div class="form-group">
            <label>Enter a unique room code: </label>
            <input asp-for="RoomCode" asp-action="CreateRoom" class="form-control"/>
        </div>

        <button type="submit" class="btn btn-primary">Submit</button>
    </form>
</body>
{% endhighlight %}

![Creating a room](https://i.imgur.com/2W1Cg7p.png)

### Joining a Room

To join a room, users simply have to enter the code for an existing room. Once again, we re-render the view if there are errors such as the user not inputting a room code or inputting a code for a non-existant room. If the user enters a valid room code, then we proceed to the EnterName view, just like we did in the CreateRoom action.

{% highlight csharp%}
[HttpGet]
public IActionResult JoinRoom()
{
    return View();
}

[HttpPost]
public IActionResult JoinRoom(JoinRoomViewModel vm)
{
    if (!ModelState.IsValid)
    {
        return View(vm);
    }

    if (Rooms.ContainsKey(vm.RoomCode))
    {
        return View(nameof(EnterName), new EnterNameViewModel { RoomCode = vm.RoomCode });
    }
    else
    {
        ModelState.AddModelError("RoomDoesNotExist", "Room does not exist");
        return View(vm);
    }
}
{% endhighlight %}

### Entering a Name

Once the user has successfully entered their name, we add them to the room's list of Gifters. At this point, we want to keep track of the user's newly generated id. One way to do that is through session storage, which we enabled back in the startup.cs file.

With the user added to the room and their id tracked in local storage, we redirect to the Room action and pass the room code as a route parameter. The reason we do this instead of just rendering the Room view is becuase we don't want users to hit the EnterName action again when they refresh the page to see if anyone has joined the room.

{% highlight csharp%}
[HttpPost]
public IActionResult EnterName(EnterNameViewModel vm)
{
    if (!ModelState.IsValid)
    {
        return View(vm);
    }


    Gifter newGifter = new Gifter(Guid.NewGuid(), vm.GifterName);
    HttpContext.Session.SetString("UserID", newGifter.ID.ToString());

    Rooms[vm.RoomCode].Gifters.Add(newGifter);

    return RedirectToAction(nameof(Room),  new { roomCode = vm.RoomCode });
}
{% endhighlight %}

{% highlight html%}
<head>
    <link rel="stylesheet" href="~/css/InputError.css" />
    <link rel="stylesheet" href="~/css/christmas.css" />
    <meta http-equiv="refresh" content="5" />
</head>
<body>
    <h1>People in the room:</h1>

    <ul>
        @foreach(Gifter gifter in Model.Room.Gifters)
        {
            <li>@gifter.Name</li>
        }
    </ul>

    <a asp-action="Recipient" asp-route-roomID="@Model.RoomCode">Get your Secret Santa recipient</a>
</body>
{% endhighlight %}

![Room View](https://i.imgur.com/yOZa8YO.png)

## Assigning Recipients

In the Recipient action, we finally assign gift recipients, after which we use session storage again to get the user associated with the current session. We then render the Recipient view, passing in the user's recipient name.

{% highlight csharp%}
[HttpGet]
public IActionResult Recipient(string roomID)
{ 
    var room = Rooms[roomID];
    room.GiftRecipients();

    var gifter = room.Gifters.FirstOrDefault(g => g.ID.ToString() == HttpContext.Session.GetString("UserID"));

    return View(new RecipientViewModel(gifter.RecipientName));
}
{% endhighlight %}


{% highlight csharp%}
public class SecretSantaRoom
{
    public List<Gifter> Gifters { get; } = new List<Gifter>();

    private bool recipientsAssigned = false;

    public void GiftRecipients()
    {
        if (!recipientsAssigned)
        {
            recipientsAssigned = true;

            Gifters.Shuffle();

            for(int i = 0; i < Gifters.Count; ++i)
            {
                //Wrap around list like circular array, assigning each gifter the next gifter in the list as their recipient
                string recipientName = Gifters[(i + 1) % Gifters.Count].Name;
                Gifters[i].RecipientName = recipientName;
            }
        }
    }
}
{% endhighlight %}

![Recipient View](https://i.imgur.com/cfiornY.png)

And with that, we've reached basic functionality. If you want to see the website in action, visit using this link and try it out yourself: [https://djb-secretsanta.herokuapp.com/](https://djb-secretsanta.herokuapp.com/)
