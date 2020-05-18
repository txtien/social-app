# Sharing Content on Your Website

# Creating an image bookmarking website

1. Define a model to store images and their information
2. Create a form and a view to handle image uploads
3. Build a system for users to be able to post images that they find on external websites

- django-admin startapp images
  INSTALLED_APPS = [

# ...

'images.apps.ImagesConfig',
]

## Building the image model

- `images/models.py`

```python
from django.db import models
from django.conf import settings

# Create your models here.
class Image(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, related_name='images_created', on_delete=models.CASCADE)
    title = models.CharField(max_length=200)
    slug = models.CharField(max_length=200, blank=True)
    url = models.URLField()
    image = models.ImageField(upload_to='image/%Y/%m/%d')
    descriptions = models.TextField(blank=True)
    created = models.DateField(auto_now_add=True, db_index=True)

    def __str__(self):
        return self.title
```

`db_index=True` so that Django creates an index in the database for this field

- You will override the save() method of the Image model to automatically generate the slug field based on the value of the title field

```python
from django.utils.text import slugify

class Image(models.Model):
# ...
def save(self, *args, **kwargs):
    if not self.slug:
        self.slug = slugify(self.title)
    super().save(*args, **kwargs)
```

## Creating many-to-many relationships

- A user might like multiple images and each image can be liked by multiple users  
  `users_like = models.ManyToManyField(settings.AUTH_USER_MODEL, related_name='images_liked', blank=True)`
- `ManyToManyField` fields provide a many-to-many manager that allows you to retrieve related objects, such as image.users_like.all(), or get them from a user object, such as user.images_liked.all().
  [Many to many relationship](https://docs.djangoproject.com/en/3.0/topics/db/examples/many_to_many/)
- Make migrations and migrate

## Registering the image model in the administration site

```python
# images/admin.py
from django.contrib import admin
from .models import Image

@admin.register(Image)
class ImageAdmin(admin.ModelAdmin):
    list_display = ['title', 'slug', 'image', 'created']
    list_filter = ['created']

```

# Posting content from other websites

- `images/forms.py`

```python
from django import forms
from .models import Image

class ImageCreateForm(forms.ModelForm):
    class Meta:
        model = Image
        fields = ('title', 'url', 'description')
        widgets = {
            'url': forms.HiddenInput
        }
```

- User will not enter image URL. Instead, you provide them with a JavaScript tool to choose an image from an external site, and your form will receive its URL as a parameter. You override the default widget of the url field to use a HiddenInput widget. This widget is rendered as an HTML input element with a type="hidden" attribute. You use this widget because you don't want this field to be visible to users

## Cleaning form fields

- In order to verify that the provided image URL is valid, you will check that the filename ends with a .jpg or .jpeg extension to only allow JPEG files

```python
#ImageCreateForm
def clean_url(self):
        url = self.cleaned_data['url']
        valid_extensions = ['jpg', 'jpeg']
        extension = url.rsplit('.', 1)[1].lower()
        if extension not in valid_extensions:
            raise forms.ValidationError('The given URL does not match valid image extensions.')
        return url
```

- Django allows you to define form methods to clean specific fields using the clean\_<fieldname>() convention. This method is executed for each field, if present, when you call is_valid() on a form instance.

## Overriding the save() method of a ModelForm

- ModelForm provides a save() method to save current model instance to db and return object. If commit=False save() method returns model instance but not save it to the database

- You will override the save() method of your form in order to retrieve the given image and save it.
- Add the following imports at the top of the forms.py file:

```python
from urllib import request
from django.core.files.base import ContentFile
from django.utils.text import slugify
```

- Then, add the following save() method to the ImageCreateForm form:

```python
def save(self, force_insert=False, force_update=False, commit=True):
    image = super().save(commit=False)
    image_url = self.cleaned_data['url']
    name = slugify(image.title)
    extension = image_url.rsplit('.', 1)[1].lower()
    image_name = f'{name}.{extension}'

    #download image from the given url
    response = request.urlopen(image_url)
    image.image.save(image_name, ContentFile(response.read()), save=False)

    if commit:
        image.save()
    return image
```

- In order to use the urllib to retrieve images from URLs served through HTTPS, you need to install the Certifi Python package. Certifi is a collection of root certificates for validating the trustworthiness of SSL/TLS certificates: pip install --upgrade certifi

- `images/views.py`

```python
from django.contrib.auth.decorators import login_required
from django.contrib import messages
from .forms import ImageCreateForm


# Create your views here.
@login_required
def image_create(request):
    if request.method == "POST":
        # form is sent
        form = ImageCreateForm(data = request.POST)
        if form.is_valid():
            cd = form.cleaned_data
            new_item = form.save(commit=False)

            # assign current user to item
            new_item.user = request.user
            new_item.save()
            messages.success(request, 'Image added successfully!')
            return redirect(new_item.get_absolute_url())
        else:
            # build form with data provided by the bookmarklet via GET
            form = ImageCreateForm(data=request.GET)
    return render(request, 'images/image/create.html', {'section': 'images', 'form': form})
```

- `images/urls.py`

```python
from django.urls import path
from . import views
app_name = 'images'
urlpatterns = [
    path('create/', views.image_create, name='create'),
]

```

## Building a bookmarklet with jQuery

- A bookmarklet is a bookmark stored in a web browser that contains JavaScript code to extend the browser's functionality.
- Some online services, such as Pinterest, implement their own bookmarklets to let users share content from other sites onto their platform.
- Let's create a bookmarklet in a similar way for your website, using jQuery to build your bookmarklet.

- This is how your users will add a bookmarklet to their browser and use it:

  1. The user drags a link from your site to their browser's bookmarks. The link contains JavaScript code in its href attribute. This code will be stored in the bookmark.
  2. The user navigates to any website and clicks on the bookmark. The JavaScript code of the bookmark is executed.

- Create new file at images/templates/ and name it `bookmarklet_launcher.js`

```javascript
(function () {
  if (window.myBookmarklet !== undefined) {
    myBookmarklet();
  } else {
    document.body.appendChild(document.createElement("script")).src =
      "https://127.0.0.1:8000/static/js/bookmarklet.js?r=" +
      Math.floor(Math.random() * 99999999999999999999);
  }
})();
```

- Edit all `account/dashboard.html`

```html
{% extends "base.html" %}
{% block title %}Dashboard{% endblock %}
{% block content %}
        <h1>Dashboard</h1>
        {% with total_images_created=request.user.images_created.count %}
                <p>Welcome to your dashboard. You have bookmarked {{total_images_created}} image{{total_images_created|pluralize}}.</p>
        {%endwith%}
        <p>Drag the following button to your bookmarks toolbar to bookmark images from other websites → <a
                        href="javascript:{% include"bookmarklet_launcher.js" %}" class="button">Bookmark it</a></p>
        <p>You can also <a href="{% url "edit" %}">edit your profile</a>or <a href="{% url "password_change" %}">change your password</a>.</p>
{% endblock %}
```

- Create images/templates/bookmarket_launcher.js, images/static/bookmarklet.js

# Creating a detail view for images

- `images/views.py`

```python
def image_detail(request, id, slug):
    image = get_object_or_404(Image, id=id, slug=slug)
    return render(request, 'images/image/detail.html', {'section': 'images', 'image': image})
```

- `images/urls.py`

```python
urlpatterns = [
    path('create/', views.image_create, name='create'),
    path('detail/<int:id>/<slug:slug>/', views.image_detail, name='detail')
]
```

- `images/models.py`

```python
class Image(models.Model)
    # ...
    def get_absolute_url(self):
        return reverse('images:detail', args=[self.id, self.slug])
```

- Create `images/templates/images/image/detail.html`

```html
{% extends "base.html" %} {% block title %}{{image.title}}{% endblock %} {%
block content %}
<h1>{{ image.title }}</h1>
<img src="{{ image.image.url }}" class="image-detail" />
{% with total_likes=image.users_like.count %}
<div class="image-info">
  <div>
    <span class="count">
      {{ total_likes }} like{{ total_likes|pluralize }}
    </span>
  </div>
  {{ image.description|linebreaks }}
</div>
<div class="image-likes">
  {% for user in image.users_like.all %}
  <div>
    <img src="{{ user.profile.photo.url }}" />
    <p>{{ user.first_name }}</p>
  </div>
  {% empty %} Nobody likes this image yet. {% endfor %}
</div>
{% endwith %} {% endblock %}
```

# Creating image thumbnails using easy-thumbnails

- Display optimized images in a thumbnail way
- pip install easy-thumbnails==2.7

INSTALLED_APPS = [

# ...

'easy_thumbnails',
]

- Migrate
- The application provides a `{% thumbnail %}` template tag to generate thumbnails in templates and a custom ImageField if you want to define thumbnails in your models.
- Edit `image/detail.html`, replace <img src="{{ image.image.url }}" class="image-detail">:

```html
{% load thumbnail %}
<a href="{{ image.image.url }}">
  <img src="{% thumbnail image.image 300x0 %}" class="image-detail" />
</a>
```

- You define a thumbnail with a fixed width of 300 pixels and a flexible height to maintain the aspect ratio by using the value 0
- You can use a different quality value using the quality parameter. To set the highest JPEG quality, you can use the value 100, like this {% thumbnail image.image 300x0 quality=100 %}.

# Adding AJAX actions with jQuery

- You are going to add a link to the image detail page to let users click on it in order to like an image. You will perform this action with an AJAX call to avoid reloading the whole page.
- Edit `images/views.py`

```python
from django.http import JsonResponse
from django.views.decorators.http import require_POST

@login_required
@require_POST
def image_like(request):
    image_id = request.POST.get('id')
    action = request.POST.get('action')
    if image_id and action:
        try:
        image = Image.objects.get(id=image_id)
            if action == 'like':
                image.users_like.add(request.user)
            else:
                image.users_like.remove(request.user)
            return JsonResponse({'status':'ok'})
        except:
            pass
    return JsonResponse({'status':'error'})
```

- `images/urls.py`

```python
path('like/', views.image_like, name='like'),
```

## Loading jQuery

- Edit `base.html` before closing </body>:

```javascript
<script src="https://ajax.googleapis.com/ajax/libs/jquery/3.4.1/jquery.min.js"></script>
  <script>
    $(document).ready(function () {
      {% block domready %}
      {% endblock %}
    });
  </script>
```

- \$(document).ready() is a jQuery function that takes a handler that is executed when the Document Object Model (DOM) hierarchy has been fully constructed.

## Cross-site request forgery in AJAX requests

- With CSRF protection active, Django checks for a CSRF token in all POST requests. When you submit forms, you can use the `{% csrf_token %}` template tag to send the token along with the form. However, it is a bit inconvenient for AJAX requests to pass the CSRF token as POST data with every POST request
- Django allows you to set a custom X-CSRFToken header in your AJAX requests with the value of the CSRF token
  [CSRF AJAX](https://docs.djangoproject.com/en/3.0/ref/csrf/#ajax)

- There are 2 steps:
  - Retrieve the CSRF token from the csrftoken cookie
  - Send the token in the AJAX request using the X-CSRFToken header

* Edit script in `base.html`

```javascript
<script src="https://cdn.jsdelivr.net/npm/js-cookie@2.2.1/src/js.cookie.min.js"></script>
<script>
var csrftoken = Cookies.get('csrftoken');
function csrfSafeMethod(method) {
    // these HTTP methods do not require CSRF protection
    return (/^(GET|HEAD|OPTIONS|TRACE)$/.test(method));
}
$.ajaxSetup({
    beforeSend: function (xhr, settings) {
    if (!csrfSafeMethod(settings.type) && !this.crossDomain) {
        xhr.setRequestHeader("X-CSRFToken", csrftoken);
    }
    }
});
$(document).ready(function () {
    // ...
}
```

## Performing AJAX requests with jQuery

- Edit `images/image/detail.html`, replace {% with total_likes=image.users_like.count %}

```html
{% with total_likes=image.users_like.count users_like=image.users_like.all %}

<!-- Replace for loop -->
{% for user in users_like %}

<!-- Modify image-info tag -->
<div class="image-info">
  <div>
    <span class="count">
      <!-- This line -->
      <span class="total">{{ total_likes }}</span>
      like{{ total_likes|pluralize }}
    </span>
    <!-- This whole <a></a> tag -->
    <a
      href="#"
      data-id="{{ image.id }}"
      data-action="{% if request.user in users_like %}un{% endif %}like"
      class="like button"
    >
      {% if request.user not in users_like %} Like {% else %} Unlike {% endif %}
    </a>
  </div>
  {{ image.description|linebreaks }}
</div>
```

- WHen user click on like/unlike link, you will perform the following
  actions on the client side:
  1 .Call the AJAX view, passing the image ID and the action parameters to it

2. If the AJAX request is successful, update the data-action attribute of the<a> HTML element with the opposite action (like / unlike), and modify its display text accordingly
3. Update the total number of likes that is displayed

# Creating custom decorators for your views

- Let's restrict your AJAX views to allow only requests generated via AJAX. The Django request object provides an `is_ajax()`
- Create bookmark/common folder, create 2 files **init**.py and decorators.py

```python
#decorators.py
from django.http import HttpResponseBadRequest

def ajax_required(f):
    def wrap(request, *args, **kwargs):
        if not request.is_ajax():
            return HttpResponseBadRequest()
        return f(request, *args, **kwargs)
    wrap.__doc__=f.__doc__
    wrap.__name__=f.__name__
    return wrap
```

- Edit `images/views.py`, image_like function

```python
from bookmarks.common.decorators import ajax_required
@ajax_required
@login_required
# ...

```

# Adding AJAX pagination to your list views

- Use AJAX pagination to build an infinite scroll functionality. Infinite scroll is achieved by loading the next results automatically when the user scrolls to the bottom of the page.
- When the user initially loads the image list page, you will display the first page of images. When they scroll to the bottom of the page, you will load the following page of items via AJAX and append it to the bottom of the main page

- Edit `images/views.py`

```python
@login_required
def image_list(request):
    images = Image.objects.all()
    paginator = Paginator(images, 8)
    page = request.GET.get('page')
    try:
        images = paginator.page(page)
    except PageNotAnInteger:
        # If page is not an integer deliver the first page
        images = paginator.page(1)
    except EmptyPage:
        if request.is_ajax():
            # If the request is AJAX and the page is out of range
            # return an empty page
            return HttpResponse('')
        # If page is out of range deliver last page of results
        images = paginator.page(paginator.num_pages)
    if request.is_ajax():
        return render(request, 'images/image/list_ajax.html', {'section': 'images', 'images': images})
    return render(request, 'images/image/list.html', {'section': 'images', 'images': images})
```

- For AJAX requests, you render the list_ajax.html template. This template will only contain the images of the requested page.
- For standard requests, you render the list.html template. This template will extend the base.html template to display the whole page and will include the list_ajax.html template to include the list of images.

- `images/urls.py`

```python
path('', views.image_list, name='list'),
```

- Create `images/image/list_ajax.html`:

```html
{% load thumbnail %} {% for image in images %}
<div class="image">
  <a href="{{image.get_absolute_url}}">
    {% thumbnail image.image 300x300 crop="smart" as im %}
    <a href="{{image.get_absolute_url" }}>
      <img src="{{im.url}}" alt="" />
    </a>
  </a>
  <div class="info">
    <a href="{{image.get_absolute_url}}" class="title">
      {{image.title}}
    </a>
  </div>
</div>
{%endfor%}
```

- Create `images/image/list.html`

```html
{% extends "base.html" %} 
{% block title %}Images bookmarked{% endblock %} {%
block content %}
<h1>Images bookmarked</h1>
<div id="image-list">
  {% include "images/image/list_ajax.html" %}
</div>
{% endblock %}


{% block domready %}
    var page = 1
    var empty_page = false
    var block_request = false

    $(window).scroll(function(){
        var margin = $(document).height() - $(window).height() - 200
        if ($(window).scrollTop() > margin && empty_page == false && block_request == false){
            block_request = true
            page += 1
            $.get('?page=' + page, function(data){
                if (data == ''){
                    empty_page = true
                } else {
                    block_request = false
                    $("#image-list").append(data)
                }
            })
        }
    })
{% endblock %}
```
- page: Stores the current page number.
- empty_page: Allows you to know whether the user is on the last page and retrieves an empty page. As soon as you get an empty page, you will stop sending additional AJAX requests because you will assume that there are no more results.
- block_request: Prevents you from sending additional requests while an AJAX request is in progress.
- You calculate the margin variable to get the difference between the total document height and the window height, because that's the height of the remaining content for the user to scroll. You subtract a value of 200 from the result so that you load the next page when the user is closer than 200 pixels to the bottom of the page.
- You perform an AJAX GET request using $.get() and receive the HTML response in a variable called data. The following are the two scenarios:
    - The response has no content: You got to the end of the results, and there are no more pages to load. You set empty_page to true to prevent additional AJAX requests.
    - The response contains data: You append the data to the HTML element with the image-list ID. The page content expands vertically, appending results when the user approaches the bottom of the page. 

- Make image-list scrollable (add display grid)