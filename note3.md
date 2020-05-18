# Tracking User Actions

# Building a follow system

- This means that your users will be able to follow each other and track what other users share on the platform.

## Creating many-to-many relationships with an intermediary model

- Creating an intermediary model is necessary when you want to store additional information for the relationship, for example, the date when the relationship was created, or a field that describes the nature of the relationship
- Let's create an intermediary model to build relationships between users. There are two reasons for using an intermediary model:

  - You are using the User model provided by Django and you want to avoid altering it
  - You want to store the time when the relationship was created

- Edit `account/models.py`

```python
from django.contrib.auth import get_user_model

# Create your models here.

user_model = get_user_model()
user_model.add_to_class('following', models.ManyToManyField('self', through=Contact, related_name='followers', symmetrical=False))

class Contact(models.Model):
    user_from = models.ForeignKey('auth.User', related_name='rel_from_set', on_delete=models.CASCADE)
    user_to = models.ForeignKey('auth.User', related_name='rel_to_set', on_delete=models.CASCADE)
    created = models.DateTimeField(auto_now_add=True, db_index=True)

    class Meta:
        ordering = ('-created',)

    def __str__(self):
        return f"{self.user_from} follows {self.user_to}"

```

user1 = User.objects.get(id=1)
user2 = User.objects.get(id=2)
Contact.objects.create(user_from=user1, user_to=user2)

- Because you don't have class User, you'll use `get_user_model` and `add_to_class`
- You tell Django to use your custom intermediary model for the relationship by adding `through=Contact` to the ManyToManyField. This is a many-to-many relationship from the User model to itself; you refer to `self` in the ManyToManyField field to create a relationship to the same model.
- You retrieve related objects using the Django ORM with user.followers.all() and user.following.all()
- Note that the relationship includes `symmetrical=False`. When you define a ManyToManyField in the model creating a relationship with itself, Django forces the relationship to be symmetrical. In this case, you are setting `symmetrical=False` to define a non-symmetrical relationship (if they follow you, it doesn't mean that you automatically follow them).

## Creating list and detail views for user profiles

- Edit `account/views.py`

```python
@login_required
def user_list(request):
    users = User.objects.filter(is_active=True)
    return render(request, 'account/user/list.html', {'section': 'people', 'users': users})

@login_required
def user_detail(request, username):
    user = get_object_or_404(User, username=username, is_active=True)
    return render(request, 'account/user/detail.html', {'section': 'people', 'user': user})
```

- `account/urls.py`

```python
urlpatterns = [
# ...
path('users/', views.user_list, name='user_list'),
path('users/<username>/', views.user_detail, name='user_detail'),
]
```

- Want to display user detail, you define get_absolute_url(). Another way to specify the URL for a model is by adding the ABSOLUTE_URL_OVERRIDES setting to your project
- `settings.py`

```python
from django.urls import reverse_lazy
ABSOLUTE_URL_OVERRIDES = {
'auth.user': lambda u: reverse_lazy('user_detail',
args=[u.username])
}
```

Now, you can use get_absolute_url() on a User instance to retrieve its corresponding URL.

```shell
>>> from django.contrib.auth.models import User
>>> user = User.objects.latest('id')
>>> str(user.get_absolute_url())
'/account/users/ellington/'
```

- In account/templates/account/user, create list.html and detail.html

```html
<!-- list.html -->
{% extends "base.html" %} {% load thumbnail %} {% block title %}People{%
endblock %} {% block content %}
<h1>People</h1>
<div id="people-list">
  {% for user in users %}
  <div class="user">
    <a href="{{ user.get_absolute_url }}">
      <img src="{% thumbnail user.profile.photo 180x180 %}" />
    </a>
    <div class="info">
      <a href="{{ user.get_absolute_url }}" class="title">
        {{ user.get_full_name }}
      </a>
    </div>
  </div>
  {% endfor %}
</div>
{% endblock %}
```

- Fix this in `base.html`

```html
<li {% if section == "people" %}class="selected"{% endif %}>
    <a href="{% url "user_list" %}">People</a>
</li>
```

- `detail.html`

```html
{% extends "base.html" %} {% load thumbnail %} {% block title %}{{
user.get_full_name }}{% endblock %} {% block content %}
<h1>{{ user.get_full_name }}</h1>
<div class="profile-info">
  <img src="{% thumbnail user.profile.photo 180x180 %}" class="user-detail" />
</div>
{% with total_followers=user.followers.count %}
<span class="count">
  <span class="total">{{ total_followers }}</span>
  follower{{ total_followers|pluralize }}
</span>
<a
  href="#"
  data-id="{{ user.id }}"
  data-action="{% if request.user in user.followers.all %}un{% endif %}follow"
  class="follow
    button"
>
  {% if request.user not in user.followers.all %} Follow {% else %} Unfollow {%
  endif %}
</a>
<div id="image-list" class="image-container">
  {% include "images/image/list_ajax.html" with images=user.images_created.all
  %}
</div>
{% endwith %} {% endblock %}
```

## Building an AJAX view to follow users

- `account/views.py`

```python
@ajax_required
@require_POST
@login_required
def user_follow(request):
    user_id = request.POST.get('id')
    action = request.POST.get('action')
    if user_id and action:
        try:
            user = User.objects.get(id=user_id)
            if action == 'follow':
                Contact.objects.get_or_create(user_form=request.user, user_to=user)
            else:
                Contact.objects.filter(user_form=request.user, user_to=user).delete()
            return JsonResponse({'status': 'ok'})
        except User.DoesNotExist:
            return JsonResponse({'status': 'error'})
    return JsonResponse({'status':'error'})
```

- `account/urls.py`

```python
path('users/follow/', views.user_follow, name='user_follow'),
```

- Ensure that you place the preceding pattern before the `user_detail` URL pattern. Otherwise, any requests to /users/follow/ will match the regular expression of the user_detail pattern and that view will be executed instead

- Add dom block `account/user/detail.html`

```javascript
{% block domready %}
    $('a.follow').click(function(e){
        e.preventDefault();
        $.post('{% url "user_follow" %}',
            {id: $(this).data('id'), action: $(this).data('action')},
            function(data){
                if (data['status'] == 'ok') {
                    var previous_action = $('a.follow').data('action');
                    // toggle data-action
                    $('a.follow').data('action', previous_action == 'follow' ? 'unfollow' : 'follow');
                    // toggle link text
                    $('a.follow').text(previous_action == 'follow' ? 'Unfollow' : 'Follow');
                    // update total followers
                    var previous_followers = parseInt($('span.count .total').text());
                    $('span.count .total').text(previous_action == 'follow' ? previous_followers + 1 : previous_followers - 1);
                }
            }
        );
    });
{% endblock %}
```

# Building a generic activity stream application
- Many social websites display an activity stream to their users so that they can track what other users do on the platform.
- python manage.py startapp actions, add app in INSTALLED_APPS

- `action/models.py`
```python
from django.db import models

# Create your models here.
class Action(models.Model):
    user = models.ForeignKey('auth.User', related_name='actions', db_index=True, on_delete=models.CASCADE)
    verb = models.CharField(max_length=255)
    created = models.DateTimeField(auto_now_add=True, db_index=True)

    class Meta:
        ordering = ('-created')
```
- With this basic model, you can only store actions, such as user X did something. You need an extra ForeignKey field in order to save actions that involve a target object, such as user X bookmarked image Y or user X is now following user Y. you will need a way for the action's target object to be an instance of an existing model. This is what the Django contenttypes framework will help you to do.

## Using the contenttypes framework
- The django.contrib.contenttypes application is included in the INSTALLED_APPS setting by default
- The contenttypes application contains a ContentType model. Instances of this model represent the actual models of your application, and new instances of ContentType are automatically created when new models are installed in your project
- The ContentType model has the following fields:

    - app_label: This indicates the name of the application that the model belongs to. This is automatically taken from the app_label attribute of the model Meta options. For example, your Image model belongs to the images application.
    - model: The name of the model class.
    - name: This indicates the human-readable name of the model. This is automatically taken from the verbose_name attribute of the model Meta options.
- Example
```shell
>>> from django.contrib.contenttypes.models import ContentType
>>> image_type = ContentType.objects.get(app_label='images',
model='image')
>>> image_type
<ContentType: images | image>

>>> image_type.model_class()
<class 'images.models.Image'>
```

## Adding generic relations to your models

- In generic relations, ContentType objects play the role of pointing to the model used for the relationship. You will need three fields to set up a generic relation in a model:
    - A ForeignKey field to ContentType: This will tell you the model for the relationship
    - A field to store the primary key of the related object: This will usually be a PositiveIntegerField to match Django's automatic primary key fields
    - A field to define and manage the generic relation using the two previous fields: The contenttypes framework offers a GenericForeignKey field for this purpose

- Edit `actions/models.py`
```python
from django.contrib.contenttypes.models import ContentType
from django.contrib.contenttypes.fields import GenericForeignKey

target_ct = models.ForeignKey(ContentType, blank=True, null=True, related_name='target_obj', on_delete=models.CASCADE)
target_id = models.PositiveIntegerField(null=True, blank=True, db_index=True)
target = GenericForeignKey('target_ct','target_id')
```
- target_ct: A ForeignKey field that points to the ContentType model
- target_id: A PositiveIntegerField for storing the primary key of the related object
- target: A GenericForeignKey field to the related object based on the combination of the two previous fields
- Both fields have blank=True and null=True attributes, so that a target object is not required when saving Action objects.

- Edit `action.admin.py`
```python
from django.contrib import admin
from .models import Action
# Register your models here.

@admin.register(Action)
class ActionAdmin(admin.ModelAdmin):
    list_display = ('user', 'verb', 'target', 'created')
    list_filter = ('created',)
    search_fields = ('verb',)
```
- Go to admin page, you can see only the `target_ct` and `target_id` fields that are mapped to actual database fields are shown. The `GenericForeignKey` field does not appear in the form. The target_ct field allows you to select any of the registered models of your Django project. You can restrict the content types to choose from a limited set of models using the limit_choices_to attribute in the target_ct

- Create `actions/utils.py`
```python
from django.contrib.contenttypes.models import ContentType
from .models import Action
def create_action(user, verb, target=None):
    action = Action(user=user, verb=verb, target=target)
    action.save()
```

## Avoiding duplicate actions in the activity stream
- Sometimes, your users might click several times on the LIKE or UNLIKE button or perform the same action multiple times in a short period of time. This will easily lead to storing and displaying duplicate actions. To avoid this, let's improve the create_action()
- Edit the `utils.py` file of the actions 
```python
def create_action(user, verb, target=None):
    # check for any similar action made in the last minute
    now = timezone.now()
    last_minute = now - datetime.timedelta(seconds=60)
    similar_actions = Action.objects.filter(user_id=user.id, verb=verb, created__gte=last_minute)

    if target:
        target_ct = ContentType.objects.get_for_model(target)
        similar_actions = similar_actions.filter(target_ct=target_ct, target_id=target.id)

    if not similar_actions:
        # no existing actions found
        action = Action(user=user, verb=verb, target=target)
        action.save()
        return True
    return False
```

## Adding user actions to the activity stream
- A user bookmarks an image
- A user likes an image
- A user creates an account
- A user starts following another user

- Edit the `views.py` file of the images:
```python
from actions.utils import create_action

# In the image_create view
new_item.save()
create_action(request.user, 'bookmarked image', new_item)

# In the image_like view
image.users_like.add(request.user)
create_action(request.user, 'likes', image)
```

- Edit the `views.py` file of the account:
```python
from actions.utils import create_action

# In the register view
Profile.objects.create(user=new_user)
create_action(new_user, 'has created an account')

# In the user_follow view
Contact.objects.get_or_create(user_from=request.user, user_to=user)
create_action(request.user, 'is following', user)

```

## Displaying the activity stream
- Edit the views.py file of the account , `dashboard` view
```python
from actions.models import Action

@login_required
def dashboard(request):
    # Display all action by default
    actions = Action.objects.exclude(user=request.user)
    following_ids = request.user.following.values_list('id', flat=True)

    if following_ids:
        # If user is following others, retrieve only their actions
        actions = actions.filter(user_id__in=following_ids)
    actions = actions[:10]
    return render(request, 'account/dashboard.html', {'section': 'dashboard', 'actions': actions})
```

# Optimizing QuerySets that involve related objects
- Every time you retrieve an Action object, you will usually access its related User object and the user's related Profile object

## Using select_related()
- Django offers a QuerySet method called select_related() that allows you to retrieve related objects for one-to-many relationships.
-  The select_related method is for `ForeignKey` and `OneToOne` fields. It works by performing a SQL `JOIN` and including the fields of the related object in the `SELECT` statement

- Replace `actions = actions[:10]` to 
```python
actions = actions.select_related('user', 'user__profile')[:10]
```
- You use `user__profile` to join the Profile table in a single SQL query. If you call `select_related()` without passing any arguments to it, it will retrieve objects from `all` ForeignKey relationships

## Using prefetch_related()
- `select_related()` doesn't work for many- to-many or many-to-one relationships (ManyToMany or reverse ForeignKey fields). 
-  `prefetch_related` works for many-to-many and many-to-one relationships in addition to the relationships supported by `select_related()`, also supports the prefetching of GenericRelation and GenericForeignKey. 