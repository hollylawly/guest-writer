---
layout: post
title: "Django Authentication"
description: "Learn how to add authentication to your Django application with Auth0."
date: 2020-10-08 8:30
category: Developers, Tutorial, Django
author:
  name: "Anthony Herbert"
  url: "https://twitter.com/prettyprinted"
  avatar: "https://www.gravatar.com/avatar/76ea40cbf67675babe924eecf167b9b8?s=60"
design:
  image:
tags:
- django
- python
- authentication
- auth0
related:
- 

---
layout: post
title: "Django Authentication Guide"
metatitle: "Django Authentication Guide"
description: ""
metadescription: ""
date: 2020-10-02 09:00
category: Developers, Tutorial, Django
post_length: 1
community_topic_id: 
author_type: Auth0 Employee
author:
 name: Anthony Herbert
 url: https://prettyprinted.com
 avatar: 
design:
 illustration: https://cdn.auth0.com/blog/phishing/phishing-hero.jpg
tags:
- django
- python
- security
- auth0
- authentication
related:
- 2020-07-01-the-11-biggest-data-breaches-of-2020-so-far
- 2020-06-17-data-privacy-vs-data-security-why-your-business-needs-both
- 2020-05-27-phishing-goes-viral-covid-19-themes-in-cybercrime
excerpt: none
---

In this article you will learn how to use Auth0 for authentication in a Django app.

## Django Authentication vs Auth0

How does the default authentication for Django compare with Auth0? For one, the default authentication focuses on hashing passwords locally and only allowing a user to login with a username and password. For all the apps that use this type of authentication, it works fine, but Auth0 gives you some advantages over the default django authentication.

Even though Auth0 handles the actual credentials for your users, the user data is still stored in your app's database. So your user table will still have a user for each person who signs up with Auth0. But in addition to the user table, you will have three extra tables from Python social auth: associations, nonces, and user social auths. Only one table is primarily used though: user social auths. This stores both the provider information (auth0) and the access token for the user. 

So Auth0 will be the middle man between your users and whatever social auth they choose to use. Then they will be redirected to sign in with the social platform of choice (so you don't have to support their choices) and then the auth token for that user will be passed back to the app. This allows you to ignore integrating access to every social provider because Auth0 has already handled it for you, otherwise you'd have to do things like get an Oauth token for each individual social platform you want to support for auth.

If you use default process for django authentication, you will need to customize the login. Even though Django gives you the tables in the database, a way to hash passwords, and a way to associate data with a user, you will have to write this yourself. You will need to create the views and templates for the login associated pages. Also, if you want to implement alternative forms of authentication, like social login and passwordless login, you will need to add the code yourself. 

With Auth0, you simply connection your account and Auth0 will handle all of the authentication stuff for you. You just rely on Auth0 to identify your users through authentication. This obviously saves a ton of time because you don't need to write the same code over and over again. Login code is both common to every app yet easy to mess up, so by using Auth0 you will save a ton of time authenticating with your app. Also, you can change how your app handles authentication at any time through the Auth0 dashboard. For example, if you want to add a new social provider, you can do it within minutes on the dashboard instead of writing any code in your app to handle a new social login.

Even though Auth0 will handle the identity part of your app, you can still work with a user once they're identified just as if they signed up and logged in with your Django app directly. Since each user becomes a regular user, you can do things like add that particular user to a group on the admin dashboard to authorize them to use more features within your app. 

Even though you can use Auth0 to allow your users to authenticate in any way they wish, you can also continue the use the default Django authentication along side. The end result is the same with both processes: a user is created in the database and a cookie is created to represent that user. Though in this tutorial, you will use Auth0 exclusively for the app's users and keep the default authentication for the admin users.

## What You Will Build

For this tutorial, I'll assume you have experience with Django and its standard patterns. For this reason, I won't talk about common things like setting up an app, for example, since it's something you've probably done a million times already.

In this tutorial, you will build a public feed app, which allows users to post a message that is visible by everyone who comes to the app. In addition to the public facing part of the app, you will add in some moderation utilities to allow mods to both block users and hide posts from the feed. You will use Auth0 to handle the authentication and you will add the authorization handling within the app to distinguish between the actions that non-users, users, and moderators can take. You will use the default Django admin to allow admin users to promote regular users to be a moderator by adding them to a particular group.

## Requirements

This app will use:

Python 3.8
Django 3.1

But any Python version greater than 3.6 should work (because of f-strings) and any version over Django 2 should work. If you want to work with an older version of Python, anything over three should work as long you use something other than f-strings for string interpolation.

# Build the App

To start, create a directory for the project and create a virtual environment. Once the virtual environment is ready, you can activate it and install Django.

```bash
mkdir django-auth0
cd django-auth0
python -m venv env
source env/bin/activate
pip install django
```

Now with Django installed, you need to create a project using django-admin and then you can create an app within that project that will have all the functionality you create in this tutorial.

```bash
django-admin startproject feed
cd feed/
python manage.py startapp feedapp
```

Open up feed/feed/settings.py, and add the feedapp to the list of installed apps.

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'feedapp',
]
```

Before doing anything else, it's a good idea to create a custom user model so you will have the most flexibly when changing the user model in the future. Even though there won't be any of those changes in this simple example, it's always a good idea to create the custom user model before running the first migration. You can place this user model in your feedapp model because the project will be simple. In a more complicated app, you could place the user model in an app that handles only authentication.

```python
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    pass
```

Then you need to tell Django you want to use this as the user model inside of the settings.py file.

```python
AUTH_USER_MODEL = 'feed.User'
```

Now that you have the user model, you can migrate all the default tables. Then create a super user so you can use the admin dashboard later.


```bash
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
```

Then you will need to register the user model you created on the admin dashboard. Inside of feedapp/admin.py, you can register the model.

```py
from django.contrib.auth.admin import UserAdmin
from .models import User

admin.site.register(User, UserAdmin)
```

Now that the user model is set up, it's time to move on to the models you will need for the app: post and report. The post model will contain the information needed for a post and the report model will handle reports of users that moderators can respond to.

The post model will need the following data: user, text, date_posted, hidden, date_hidden and hidden_by. The user refers to who created the post, the text refers to the actual text content of the post and date_posted tracks when it was created so you can order by posts. hidden, date_hidden, and hidden_by are used only when a moderator decides to hide a post. Moderators won't be allowed to delete any data. Instead, they will only be allowed to hide users so hidden posts can be tracked and potentially unhidden.

You can go ahead and create that model now:

```py
class Post(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    text = models.CharField(max_length=280)
    date_posted = models.DateTimeField(auto_now_add=True)
    hidden = models.BooleanField(default=False)
    date_hidden = models.DateTimeField(blank=True, null=True)
    hidden_by = models.ForeignKey(settings.AUTH_USER_MODEL,on_delete=models.CASCADE, blank=True, null=True, related_name='mod_who_hid') 

    def __str__(self):
        return self.text
```

Everything here is normal. The only difference is for creating foreign keys to the user model, you need to use the settings.AUTH_USER_MODEL to reference the user model instead of importing the model directly. You need to import settings from django.conf at the top of the file.

```py
from django.conf import settings
```

The other model you need is for making reports. You only need two fields here: post and reported_by, where post refers to the post that got reported and reported_by is the user who created the report. Like the post model, you will use the settings.AUTH_USER_MODEL for the table in the foreign key for reported_by. The post's foreign key simply points to the Post model you just created.

```py
class Report(models.Model):
    reported_by = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
```

With the code for the models added, go ahead and make the migrations again and migrate them. 

```bash
python manage.py makemigrations
python manage.py migrate
```

Finally, add the models to the admin dashboard so you can view on the admin dashboard later.

```py
from .models import User, Post, Report

admin.site.register(User, UserAdmin)
admin.site.register(Post)
admin.site.register(Report)
```

Now that you have the models done, you can start the scaffolding for the views and then add some templates. You will also create URLs that will connect to the views. Start with the two views that will return HTML, the index view and the reports view, which will allow moderators to see all the posts that have been reported.

First, in views.py, you need to add the functions for those views. You can return a render function that will have template names passed to it. The templates haven't been created yet, but you can add their names anyway.

```py
from django.shortcuts import render

def index(request):
    return render(request, 'feedapp/index.html')

def reports(request):
    return render(request, 'feedapp/reports.html')
```

Now that you have the views, add the two templates that you need. Create a template directory with a feedapp directory inside of your feedapp package and then create the two files. Also, you can create a base.html template since both templates will have common elements.

{% highlight html %}
{% raw %}
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{% block title %}{% endblock %}</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bulma@0.9.0/css/bulma.min.css">
  </head>
  <body>
  <section class="section">
    <div class="container">
      {% block content %}{% endblock %}
    </div>
  </section>
  </body>
</html>
{% endraw %}
{% endhighlight %}

Note that you have both a title block and a content block inside of the base template.

Next, you need the template code for the index.html. For now, you will have just enough to display the title. You will fill in the rest later as you add functionality.

{% highlight html %}
{% raw %}
  {% extends 'feedapp/base.html' %}
  {% block title %}Home{% endblock %}
  {% block content %}
  {% endblock %}
{% endraw %}
{% endhighlight %}

For the reports:

{% highlight html %}
{% raw %}
  {% extends 'feedapp/base.html' %}
  {% block title %}Reported Posts{% endblock %}
  {% block content %}
  {% endblock %}
{% endraw %}
{% endhighlight %}

With the views created and the templates for them, the last thing you need to do is add URLs for the views.

You need to create a urls.py in your feedapp directory. And in it you need to give paths and combine them with the views.

```py
from django.urls import path
from . import views

urlpatterns = [
    path('', views.index, name='index'),
    path('reports/', views.reports, name='reports'),
]
```

Finally, you will need to modify the existing urls.py in the main project directory to include the urls from the file you just created.

```py
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('feedapp.urls')),
]
```

Now you have a chance to run the server and make sure you don't have any errors. Start the server and navigate to the two URLs you have to make sure you can see blank pages and see the titles in the title bar in the browser.

```bash
python manage.py runserver
```

Now that everything is working, you want to start adding some functionality to the app. Since you need users to create posts and make reports, you need to allow the concept of a user first. This is where Auth0 comes in. To continue, you will need an Auth0 account. You can create one here if you don't have one already.

# Creating Auth0 App

Once you are in your Auth0 account, go to 'Accounts' from the dashboard. There, click on 'Create Application.' Give your app a name, and select regular web app. With the app created, you can go to the settings tab to see the information you will need soon to connect the Django app with Auth0. Also, this is the place where you can create some callback URLs so Auth0 can communicate with the app.

For the callback URLs, you can add http://127.0.0.1:8000/complete/auth0 and http://localhost:8000/complete/auth0 to cover both forms of localhost on your machine. For the logout URL, you can just add the base URL: http://127.0.0.1:8000 or http://localhost:8000

Save the changes once you are done.

With the URLs added, you can go back to the Django app and integrate Auth0. First, you need to install a couple libraries that will handle the connection: python-jose and social-auth-django. You can install those now after stopping the app.

```bash
pip install python-jose social-auth-app-django
```

Social django will has some URLs, so you need to include them in the project's url.py file.

```py
  path('', include('social_django.urls')),
```

Since social Django creates some new migration files, you need to migrate the app.

```bash
python manage.py migrate
```

After installing, you need to change some parts of the settings.py

```py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'feedapp',
    'social_django',
]
```

```py
AUTH_USER_MODEL = 'feedapp.User'

# Auth0 settings
SOCIAL_AUTH_TRAILING_SLASH = False  # Remove trailing slash from routes
SOCIAL_AUTH_AUTH0_DOMAIN = '<YOUR-AUTH0-DOMAIN>'
SOCIAL_AUTH_AUTH0_KEY = '<YOUR-AUTH0-CLIENT-ID>'
SOCIAL_AUTH_AUTH0_SECRET = '<YOUR-AUTH0-CLIENT-SECRET>'
SOCIAL_AUTH_AUTH0_SCOPE = [
    'openid',
    'profile',
    'email'
]

AUTHENTICATION_BACKENDS = {
    'social_core.backends.auth0.Auth0OAuth2',
    'django.contrib.auth.backends.ModelBackend'
}

LOGIN_URL = '/login/auth0'
LOGIN_REDIRECT_URL = '/'
LOGOUT_REDIRECT_URL = '/'
```

You will have to replace the Auth0 domain, key, and secret settings with the values from your Auth0 account. 

Once you have those settings, you are done with the Auth0 integration. Social Django will handle everything for you. When a user logs in through Auth0, a new record will be created in the user table. That user will then forever be associated with the person who logs in with Auth0 with the same credentials. 

So to get the login working, you just need to add a link. You can add it to your homepage. First, you will have to check if the user is authenticated. If they are, you will show a logout link. If they aren't, then you will show the login link, which is the same one in your settings.

{% highlight html %}
{% raw %}
  {% block content %}
  {% if user.is_authenticated %}
  <a href="/logout/">Logout</a>
  {% else %}
  <a href="/login/auth0">Login</a>
  {% endif %}
  {% endblock %}
{% endraw %}
{% endhighlight %}


When you start the server and go to the home page, you should see a single link. Since you are not authenticated yet, you should see the Login link. Click the link and it will take your to the Auth0 universal signin. From there, you can sign in with either a Google account by default or your Auth0 account. If you want to change the methods of authentication, you can do it on the Auth0 dashboard and easily support things like Facebook login, Github login, passwordless login and more without changing anything inside of your app. Auth0 will handle everything related to identity for you. After you sign in, you should be redirected to the index and you should see the logout link instead of the login link because you are now authenticated.

Now take a look at what happened in the database. Go the admin dashboard and check out the models. Check out the User model first.

You should see two users. Your admin user and then a user for the method you just authenticated as. Because you have the email scope, you will already have an email associated with the account. You should notice there is no password available, so this user can never login through the admin dashboard for example, but that's OK because you want all of the users to authenticate through Auth0.

Now go back and check out the user social auths. There you should see the user you created. This will have a relationship to the user in the User table you just looked at, and it will also list Auth0 as the provider and give you a UID and extra data associated with the service the user authenticated through. As you can see here, Auth0 and Social Django handled everything for you. And as you saw on the index, you have an authenticated user because the logout link is visible instead of the login link. From here, you can treat the user object the same as you would had you created a native Django authentication system.

For the user to log out, you need to create another view that handles logging out. It's more than you had to do to get the user to log in, but as you will see, it's still only a few lines of code. You need to both logout the user in Django itself and then you need to tell Auth0 you want to log the user out by redirecting to a URL you will craft with some of the settings.

```py
@login_required
def logout(request):
    django_logout(request)
    domain = settings.SOCIAL_AUTH_AUTH0_DOMAIN
    client_id = settings.SOCIAL_AUTH_AUTH0_KEY
    return_to = http://127.0.0.1:8000 # this can be current domain
    return redirect(f'https://{domain}/v2/logout?client_id={client_id}&returnTo={return_to}')
```

You need to import settings and the logout function to do this. Since the view is called logout, create an alias called django_logout for this.

```py
from django.contrib.auth import logout as django_logout
from django.conf import settings
from django.contrib.auth.decorators import login_required
```

Finally, create a URL for the logout function in the app's urls.py file.

```py
  path('logout/', views.logout, name='logout'),
```

You can test the logout link now. You will know it's working when you get redirected to the homepage and you see the login link again.

Now that you have an active user you can work with, start by allowing that user to add a post. In the index template, add a form with a textbox to allow the user to add a post. Since you only want authenticated users to add a post, you want to guard them form with a check if the user is authenticated.

{% highlight html %}
{% raw %}
{% if user.is_authenticated %}
      <article class="media">
        
        <form method="POST">
  <div class="media-content">
    <div class="field">
      <p class="control">
        {% csrf_token %}
        {{ form.text }}
      </p>
    </div>
    <nav class="level">
      <div class="level-left">
        <div class="level-item">
          
        </div>
      </div>
      <div class="level-right">
        <div class="level-item">
          <button class="button is-info">Submit</button>
        </div>
      </div>
    </nav>
  </div>
</form>
</article>

{% endif %}
{% endraw %}
{% endhighlight %}

With the form created, you need to handle any data that the user submits. First, create a modelform for the Post model. Create a forms.py file in your feedapp directory, add the following:

```py
from django.forms import ModelForm

from .models import Post

class PostForm(ModelForm):
    class Meta:
        model = Post
        fields = ['text']
        widgets = {
            'text': Textarea(attrs={'class' : 'textarea', 'cols': 80, 'rows': 5}),
        }
```

Then in the views.py and in the index function, you need to add the code to handle a POST request. You can use the modelform to save the form to the database.

The only thing special you have to do here is associate the authenticated user with the post in the database. Don't commit the first save of the form, instead add the user first and then save it. After a successful save, you will redirect back to the index, which will use GET instead of POST.

```py
from django.shortcuts import render, redirect

from .forms import PostForm

def index(request):
    if request.method == 'POST':
        form = PostForm(request.POST)
        if form.is_valid():
            post = form.save(commit=False)
            post.user = request.user 
            post.save()
            return redirect('index')
    else:
        form = PostForm()

    context = {'form' : form}

    return render(request, 'feedapp/index.html', context)
```

Try submitting a few posts and then go to the admin dashboard to make sure they made it into the database. You should see your user account associated with your Auth0 login associated with the post.

With the posts in the database, you can now add them to the homepage for the global feed. To do that, you need to modify the index function again. You need to query for all the posts in the database that aren't hidden, and then pass that to the template. From there, you can display them.

```py
from .models import Post

else:
        form = PostForm()
        posts = Post.objects.filter(hidden=False).order_by('-date_posted').all()

    context = {'form' : form, 'posts' : posts}
```

{% highlight html %}
{% raw %}
{% load humanize %}

{% for post in posts %}
<article class="media">
  <figure class="media-left">
    <p class="image is-64x64">
      <img src="https://bulma.io/images/placeholders/128x128.png">
    </p>
  </figure>
  <div class="media-content">
    <div class="content">
      <p>
        <strong>{{ post.user.first_name }} {{ post.user.last_name }}</strong> <small>{{ post.date_posted|naturaltime }}</small>
        <br>
        {{ post.text }}
      </p>
    </div>
  </div>
</article>
{% endfor %}
{% endraw %}
{% endhighlight %}


Since you are using humanize to display the date, you need to both load it at the top of the template and add it to the installed apps.

```py
'django.contrib.humanize',
```

Notice that you are using the first name and last name associate with the user that logged in through Auth0.

If you log out, you will still see the posts, but you will no longer have the option to add a new post.

Now you want allow the user to delete their own posts as well. To do this, you need another view.

```py
def delete_post(request, post_id):
    # check if post belongs to user
    post = Post.objects.get(id=post_id)
    if post.user == request.user:
        post.delete()
    # remove it from the database
    # redirect back to same page
    return redirect('index')
```

And then add a URL for it:

```py
path('delete/<post_id>/', views.delete_post, name='delete_post'),
```

And then in the template, check if the post belongs to the user. If so, add a delete link.

{% highlight html %}
{% raw %}
  {% if post.user == request.user %}
  <div class="media-right">
    <a href="{% url 'delete_post' post.id %}" class="delete"></a>
  </div>
  {% endif %}
{% endraw %}
{% endhighlight %}

Try deleting one of your posts. If everything works, then it will redirect to the same page with the post you just deleted missing.

Now that you have posts on the site, you want the ability to moderate them. You will allow users to report posts made by other users, and you will then let moderators choose to block a user completely and/or hide their posts. To get started with this, create a moderator group in the admin dashboard. The moderator group will be given access to change both posts and users and the ability to view reports. You will use these permissions later in the code to make sure only moderators can perform the action. You can then go into the users table and give one or more users the moderators group.

With that, add a permission required decorator on the reports view. This will allow only members in the moderators group to see the reports.

```py
from django.contrib.auth.decorators import login_required, permission_required

@permission_required('feedapp.view_report', raise_exception=True)
```

Next, give the users the ability to report posts. You need to create a new view and an associate URL that will handle reporting a post. Then put a link on each post by someone other than the loginned in to let that user report post.

Start with the view:

```py
from .models import Post, Report

def report_post(request, post_id):
    post = Post.objects.get(id=post_id)

    report, created = Report.objects.get_or_create(reported_by=request.user, post=post)

    if created:
        report.save()

    return redirect('index')
```

You don't want to allow a user to create multiple reports for the same post, so use get_or_create on the model.

Next, create a URL for this in urls.py.

```py
path('report_post/<post_id>/', views.report_post, name='report_post'),
```

And finally, add the links in the template. You can check if the post belongs to the user and if the post has already been reported. Since you already have the if block for checking a post belongs to a user, add an else if to check if user is authenticated.

{% highlight html %}
{% raw %}
  {% elif user.is_authenticated %}
  <div class="media-right">
    <a href="{% url 'report_post' post.id %}"><span class="tag is-warning">report</span></a>
  </div>
  {% endif %}
{% endraw %}
{% endhighlight %}


After you test reporting a post, you can check the report table in the admin dashboard to see if it appears.

Now go back to the reports view. Here you can list all of the posts in the database that have been reported at least once.

```py
from django.db.models import Count
```

```py
@permission_required('feedapp.view_report', raise_exception=True)
def reports(request):
    reports = Post.objects.annotate(times_reported=Count('report')).filter(times_reported__gt=0).all()

    context = {'reports' : reports}

    return render(request, 'feedapp/reports.html', context)
```

And then in the reports.html template, you can display all the reported posts. It will be similar to how the posts are displayed on the home page. But you will also show the number of times the post has been reported.

{% highlight html %}
{% raw %}
{% for post in reports %}
<article class="media">
  <figure class="media-left">
    <p class="image is-64x64">
      <img src="https://bulma.io/images/placeholders/128x128.png">
    </p>
  </figure>
  <div class="media-content">
    <div class="content">
      <p>
        <strong>{{ post.user.first_name }} {{ post.user.last_name }}</strong> <small>{{ post.date_posted|naturaltime }}</small>
        <br>
        {{ post.text }} - Times reported: {{ post.times_reported }}
      </p>
    </div>
  </div>
</article>
{% endfor %}
{% endraw %}
{% endhighlight %}


Now that the moderators can see reported posts, you want to allow the moderator to block users and hide posts. Start with hiding posts. You can create a view that requires a change post permission. From there, you will get the post and hide it by updating the hidden, date_hidden, and hidden_by fields on the post. You will need the datetime object to get the current date.

```py
@permission_required('feedapp.change_post', raise_exception=True)
def hide_post(request, post_id):
    post = Post.objects.get(id=post_id)
    post.hidden = True
    post.date_hidden = datetime.now()
    post.hidden_by = request.user
    post.save()
    return redirect('reports')
```

Then you need a URL for this view.

```py
path('hide_post/<post_id>/', views.hide_post, name='hide_post'),
```

Finally, add the link in the template. Check if it has been hidden already so you don't hide it more than once.

{% highlight html %}
{% raw %}
  {% if not post.hidden %}
  <div class="media-right">
    <a href="{% url 'hide_post' post.id %}" class="delete"></a>
  </div>
  {% endif %}
{% endraw %}
{% endhighlight %}


After hiding a post, you can verify by checking the homepage and seeing the post is no longer there because of the query you wrote earlier to check for hidden=False.

The other thing you want a moderator to do is block a user. When you block a user, you want to both set a user to be inactive and you want to hide all of their posts. Create the view for this:

```py
from django.contrib.auth import get_user_model

@permission_required('feedapp.change_user')
def block_user(request, user_id):
	User = get_user_model()

    user = User.objects.get(id=user_id)
    for post in user.post_set.all():
        if not post.hidden:
            post.hidden = True
            post.hidden_by = request.user
            post.date_hidden = datetime.now()
            post.save()

    user.is_active = False
    user.save()

    return redirect('reports')
```

Then add a URL for this view:

```py
path('block_user/<user_id>/', views.block_user, name='block_user'),
```

And finally you can add the link on the reports template. Make sure the user hasn't been deactivated before.

{% highlight html %}
{% raw %}
  {% if post.user.is_active %}
  <a href="{% url 'block_user' post.user.id %}"><span class="tag is-warning">Block</span></a>
  {% endif %}
{% endraw %}
{% endhighlight %}

Then you can test blocking a user. You can verify all their posts are gone.

# Conclusion

That's everything we wanted for the app. You have the feed app that allows users to post posts and moderators to make sure things don't get out of handle. Here is what you covered:

- Set up Auth0 to handle Django authentication
- Use Python social apps to store your users' tokens and account information
- Create an authorization system for different user types through roles