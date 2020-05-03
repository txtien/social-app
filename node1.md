# Using Django authentication framework
- Located at django.contrib.auth
- Handle user authentication, sessions, permissions, and user groups
- Includes views for common user actions such as log in, log out, password change, and password reset.
- Authentication framework is included when `startapp`, it consists of the `django.contrib.auth` and two middleware:
    - AuthenticationMiddleware: Associates users with requests using sessions
    - SessionMiddleware : Handles the current session across requests
- Then models:
    - User : A user model with basic fields; the main fields of this model are username , password , email , first_name , last_name , and is_active
    - Group : A group model to categorize users
    - Permission : Flags for users or groups to perform certain actions
# Create Login View
- Your view should perform the following actions to `log in` a user:
    - Get username and passwork through `post` from log in form
    - Authenticate the user against the data store in db
    - Check whether the user is active
    - Log the user into the website and start `authentication session`
```python
# account/form.py
from django import forms

class LoginForm(forms.Form):
    username = forms.CharField()
    password = forms.CharField(widget=forms.PasswordInput)
```
- PasswordInput will include `type=password` in HTML

```python
#account/views.py
from django.shortcuts import render
from django.http import HttpResponse
from django.contrib.auth import authenticate, login
from .forms import LoginForm
# Create your views here.

def user_login(request):
    if request.method == 'POST':
        form = LoginForm(request.POST)
        if form.is_valid():
            cd = form.cleaned_data
            user = authenticate(request, username=cd['username'], password=cd['password'])
            if user is not None:
                if user.is_active:
                    login(request, user)
                    return HttpResponse("Authenticated Successfully")
                else:
                    return HttpResponse("Disable account")
            else:
                return HttpResponse("Invalid Login")
    else:
        form = LoginForm()
    return render(request, "account/login.html", {'form': form})                
```
- `authenticate() method` will return `User` object if the user has been successfully authenticated, or None otherwise.
- You set the user in the session by calling the `login() `method    

- Define urls.py in account and bookmarks

- Create login.html and base.html

# Using Django authentication views
- Located in django.contrib.auth.views

[More about authentication views](https://docs.djangoproject.com/en/3.0/topics/auth/default/#all-authentication-views)

## Login and logout views
```python
#account/urls.py
from django.contrib.auth import views as auth_views

urlpatterns = [
    # Comment old path
    # path('login/', views.user_login, name="login")
    path('login/', auth_views.LoginView.as_view(), name='login'),
    path('/logout', auth_views.LogoutView.as_view(), name='logout')
]
```
• LoginView : Handles a login form and logs in a user
• LogoutView : Logs out a user

- Create account/templates/registration/login.html
```html
{% extends "base.html" %}
{% block title %}Log-in{% endblock %}
{% block content %}
<h1>Log-in</h1>
{% if form.errors %}
    <p> Your username and password didn't match.
Please try again.</p>
{% else %}
    <p>Please, use the following form to log-in:</p>
{% endif %}
<div class="login-form">
    <form action="{% url 'login' %}" method="post">
        {{ form.as_p }}
        {% csrf_token %}
        <input type="hidden" name="next" value="{{ next }}" />
        <p><input type="submit" value="Log-in"></p>
    </form>
</div>
{% endblock %}
```
- You have added a hidden HTML `<input>` element to submit the value of a variable called `next` . This variable is first set by the login view when you pass a `next`
- The `next` parameter `has to be a URL`. If this parameter is given, the Django login view will redirect the user to the given URL after a successful login.

- Create registration/logged_out.html
```html
{% extends "base.html" %}
{% block title %}Logged out{% endblock %}
{% block content %}
<h1>Logged out</h1>
<p>
You have been successfully logged out.
You can <a href="{% url "login" %}">log-in again</a>.
</p>
{% endblock %}
```
- This is the template Django will use after user log out

- Edit account/views.py
```python
from django.contrib.auth.decorators import login_required

@login_required
def dashboard(request):
    return render(request, 'account/dashboard.html', {'section': 'dashboard'})
```

- `login_required` decorator checks whether the current user is authenticated. If the user is authenticated, it executes the decorated view; if the user is not authenticated, it redirects the user to the login URL with the originally requested URL as a GET parameter named next
- By doing this, the login view redirects users to the URL that they were trying to access after they successfully log in. You added a hidden input in the form of your login template for this purpose
- You also define `section` variable, use this variable to track the site's section that the user is browsing

- Create templates/account/dashboard.html
```html
{% extends "base.html" %}
{% block title %}Dashboard{% endblock %}
{% block content %}
<h1>Dashboard</h1>
<p>Welcome to your dashboard.</p>
{% endblock %}
```

- Edit account/urls.py
```python
urlpatterns = [
    ...
    path('', views.dashboard, name='dashboard')
]
```

- Edit settings.py
```python
LOGIN_REDIRECT_URL = 'dashboard'
LOGIN_URL = 'login'
LOGOUT_URL = 'logout'
```
- LOGIN_REDIRECT_URL: tells Django which URL to redirect user after login successfully if no `next` parameter in request
- LOGIN_URL : The URL to redirect the user to log in (for example, views using the login_required decorator) 
- LOGOUT_URL : The URL to redirect the user to log out