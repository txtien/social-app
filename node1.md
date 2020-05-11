# Starting

INSTALLED_APPS = [
'account.apps.AccountConfig',

# ...

]

- **Placing your app first in INSTALLED_APPS, you ensure that your authentication templates will be used by default**

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
- You set the user in the session by calling the `login()`method

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
{% extends "base.html" %} {% block title %}Log-in{% endblock %} {% block content
%}
<h1>Log-in</h1>
{% if form.errors %}
<p>Your username and password didn't match. Please try again.</p>
{% else %}
<p>Please, use the following form to log-in:</p>
{% endif %}
<div class="login-form">
  <form action="{% url 'login' %}" method="post">
    {{ form.as_p }} {% csrf_token %}
    <input type="hidden" name="next" value="{{ next }}" />
    <p><input type="submit" value="Log-in" /></p>
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
{% extends "base.html" %} {% block title %}Dashboard{% endblock %} {% block
content %}
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

- Edit `base.html`:

```html
<div id="header">
    <span class="logo">Bookmarks</span>
    {% if request.user.is_authenticated %}
        <ul class="menu">
            <li {% if section == "dashboard" %}class="selected"{% endif %}>
            <a href="{% url "dashboard" %}">My dashboard</a>
            </li>
            <li {% if section == "images" %}class="selected"{% endif %}>
            <a href="#">Images</a>
            </li>
            <li {% if section == "people" %}class="selected"{% endif %}>
            <a href="#">People</a>
            </li>
        </ul>
    {% endif %}

    <span class="user">
        {% if request.user.is_authenticated %}
            Hello {{ request.user.first_name }},
            <a href="{% url "logout" %}">Logout</a>
        {% else %}
            <a href="{% url "login" %}">Log-in</a>
        {% endif %}
    </span>
</div>
```

## Changing password views

- Edit `account/urls.py`

# change password urls

path('password_change/',
auth_views.PasswordChangeView.as_view(),
name='password_change'),

path('password_change/done/',
auth_views.PasswordChangeDoneView.as_view(),
name='password_change_done'),

- Add a new file inside the `templates/registration/`directory of your account application and name it `password_change_form.html`

```html
{% extends "base.html" %} {% block title %}Change your password{% endblock %} {%
block content %}
<h1>Change your password</h1>
<p>Use the form below to change your password.</p>
<form method="post">
  {{ form.as_p }}
  <p><input type="submit" value="Change" /></p>
  {% csrf_token %}
</form>
{% endblock %}
```

- The `password_change_form.html` template includes the form to change the password.

- Create another file `password_change_done.html`:

```html
{% extends "base.html" %} {% block title %}Password changed{% endblock %} {%
block content %}
<h1>Password changed</h1>
<p>Your password has been successfully changed.</p>
{% endblock %}
```

## Resetting password views

- Edit `account/urls.py`

# reset password urls

path('password_reset/',
auth_views.PasswordResetView.as_view(),
name='password_reset'),

path('password_reset/done/',
auth_views.PasswordResetDoneView.as_view(),
name='password_reset_done'),

path('reset/<uidb64>/<token>/',
auth_views.PasswordResetConfirmView.as_view(),
name='password_reset_confirm'),

path('reset/done/',
auth_views.PasswordResetCompleteView.as_view(),
name='password_reset_complete'),

- `templates/registration/` create `password_reset_form.html`

```html
{% extends "base.html" %} {% block title %}Reset your password{% endblock %} {%
block content %}
<h1>Forgotten your password?</h1>
<p>Enter your e-mail address to obtain a new password.</p>
<form method="post">
  {{ form.as_p }}
  <p><input type="submit" value="Send e-mail" /></p>
  {% csrf_token %}
</form>
{% endblock %}
```

- Create `password_reset_email.html`

```html
Someone asked for password reset for email {{ email }}. Follow the link below:
{{ protocol }}://{{ domain }}{% url "password_reset_confirm" uidb64=uid
token=token %} Your username, in case you've forgotten: {{ user.get_username }}
```

- Create `password_reset_done.html`:

```html
{% extends "base.html" %} {% block title %}Reset your password{% endblock %} {%
block content %}
<h1>Reset your password</h1>
<p>We've emailed you instructions for setting your password.</p>
<p>
  If you don't receive an email, please make sure you've entered the address you
  registered with.
</p>
{% endblock %}
```

- Create `password_reset_confirm.html.`

```html
{% extends "base.html" %} {% block title %}Reset your password{% endblock %} {%
block content %}
<h1>Reset your password</h1>
{% if validlink %}
<p>Please enter your new password twice:</p>
<form method="post">
  {{ form.as_p }} {% csrf_token %}
  <p><input type="submit" value="Change my password" /></p>
</form>
{% else %}
<p>
  The password reset link was invalid, possibly because it has already been
  used. Please request a new password reset.
</p>
{% endif %} {% endblock %}
```

- Check the link for resetting the password is valid
  by checking the `validlink` variable. The view `PasswordResetConfirmView` checks the validity of the token provided in the URL and passes the validlink variable to the template. If the link is valid, you display the user password reset form. Users can only set a new password if they have a valid reset password link.

- Create `password_reset_complete.html`

```html
{% extends "base.html" %}
{% block title %}Password reset{% endblock %}
{% block content %}
    <h1>Password set</h1>
    <p>Your password has been set. You can
    <a href="{% url "login" %}">log in now</a></p>
{% endblock %}
```

- Add this line after `form` element in `registration/login.html`:

```html
<p><a href="{% url "password_reset" %}">Forgotten your password?</a></p>
```

- You have now integrated the views of the Django authentication framework into your project. These views are suitable for most cases. However, you can create your own views if you need a different behavior.

# User registration and user profiles

- Edit `account/forms.py`

```python
from django.contrib.auth.models import User

class UserRegistrationForm(forms.ModelForm):
    password = forms.CharField(label='Password',
    widget=forms.PasswordInput)
    password2 = forms.CharField(label='Repeat password',
    widget=forms.PasswordInput)

    class Meta:
        model = User
        fields = ('username', 'first_name', 'email')

    def clean_password2(self):
        cd = self.cleaned_data
        if cd['password'] != cd['password2']:
            raise forms.ValidationError('Passwords don\'t match.')
        return cd['password2']
```

- You include only the `username, first_name, and email` fields of the model. These fields will be validated based on their corresponding model fields. For example, if the user chooses a username that already exists, they will get a validation error because `username` is a field defined with `unique=True`.

* Edit `views.py`:

```python
def register(request):
    if request.method == 'POST':
        user_form = UserRegistrationForm(request.POST)
        if user_form.is_valid():
            # Create a new user object but avoid saving it yet
            new_user = user_form.save(commit=False)
            # Set the chosen password
            new_user.set_password(user_form.cleaned_data['password'])
            # Save the user
            new_user.save()
            return render(request, 'account/register_done.html', {'new_user': new_user})
    else:
        user_form = UserRegistrationForm()

    return render(request, 'account/register.html', {'user_form': user_form})
```

- Edit `urls.py`:

```python
path('register/', views.register, name='register')
```

- Create `account/register.html`:

```html
{% extends "base.html" %} {% block title %}Create an account{% endblock %} {%
block content %}
<h1>Create an account</h1>
<p>Please, sign up using the following form:</p>
<form method="post">
  {{ user_form.as_p }} {% csrf_token %}
  <p><input type="submit" value="Create my account" /></p>
</form>
{% endblock %}
```

- Create `account/register_done.html`

```html
{% extends "base.html" %}
{% block title %}Welcome{% endblock %}
{% block content %}
<h1>Welcome {{ new_user.first_name }}!</h1>
<p>Your account has been successfully created. Now you can <a href="{% url "login" %}">log in</a>.</p>
{% endblock %}
```

## Extending the user model

- The best way to do this is by creating a profile model that contains all additional fields and a one-to-one relationship with the Django User model. A one-to-one relationship is similar to a ForeignKey field with the parameter `unique=True`

- Edit `account/models.py`:

```python
from django.db import models
from django.conf import settings

# Create your models here.

class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    date_of_birth = models.DateField(blank=True, null=True)
    photo = models.ImageField(upload_to='users/%Y/%m/%d/', blank=True)

    def __str__(self):
        return f'Profile for user {self.user.username}'
```

- Add these settings to `settings.py`:
  MEDIA_URL = '/media/'
  MEDIA_ROOT = os.path.join(BASE_DIR, 'media/')

`MEDIA_URL` is the base URL used to serve the media files uploaded by users, and `MEDIA_ROOT` is the local path where they reside.

- Edit `bookmark/urls.py`:

```python
...
from django.conf import settings
from django.conf.urls.static import static


urlpatterns = [
    path('admin/', admin.site.urls),
    path('account/', include('account.urls')),
]
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

- Django development server will be in charge of serving the media files during development (that is when the DEBUG setting is set to True)

- makemigrations and migrate

- Edit `admin.py`:

```python
from django.contrib import admin
from .models import Profile

# Register your models here.

@admin.register(Profile)
class ProfileAdmin(admin.ModelAdmin):
    list_display = ('user', 'date_of_birth', 'photo')
```

- Edit `forms.py`:

```python
class UserEditForm(forms.ModelForm):
    class Meta:
        model = User
        fields = ('first_name', 'last_name', 'email')

class ProfileEditForm(forms.ModelForm):
    class Meta:
        model = Profile
        fields = ('date_of_birth', 'photo')
```

- UserEditForm: This will allow users to edit their first name, last name, and email, which are attributes of the built-in Django user model.

- ProfileEditForm: This will allow users to edit the profile data that you save in the custom Profile model. Users will be able to edit their date of birth and upload a picture for their profile.

- Add following lines in `account/views.py`, at `register` def below `new_user.save()`

```python
# Create the user profile
Profile.objects.create(user=new_user)
```

When users register on your site, you will create an empty profile associated with them.

- Edit `views.py`:

```python

@login_required
def edit(request):
    if request.method == 'POST':
        user_form = UserEditForm(instance=request.user, data=request.POST)
        profile_form = ProfileEditForm(
            instance=request.user.profile, data=request.POST, files=request.FILES)

        if user_form.is_valid() and profile_form.is_valid():
            user_form.save()
            profile_form.save()
    else:
        user_form = UserEditForm(instance=request.user)
        profile_form = ProfileEditForm(instance=request.user.profile)
    return render(request, 'account/edit.html', {'user_form': user_form, 'profile_form': profile_form})
```

- Add url in `urls.py`:

```python
path('edit/', views.edit, name='edit')
```

- Create `account/edit.html`:

```html
{% extends "base.html" %} {% block title %}Edit your account{% endblock %} {%
block content %}
<h1>Edit your account</h1>
<p>You can edit your account using the following form:</p>
<form method="post" enctype="multipart/form-data">
  {{ user_form.as_p }} {{ profile_form.as_p }} {% csrf_token %}
  <p><input type="submit" value="Save changes" /></p>
</form>
{% endblock %}
```
- Edit `dashboard.html` `<p>Welcome to dashboard</p>` to: 
<p>Welcome to your dashboard. You can <a href="{% url "edit" %}">edit your profile</a> or <a
        href="{% url "password_change" %}">change your password</a>.</p>