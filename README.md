# django-autocomplete-tom-select-htmx

## Custom TomSelect field

```python
class TomSelectField(forms.CharField):
    def __init__(self, *args, **kwargs):
        self.queryset = kwargs.pop("queryset")
        self.url_name = kwargs.pop("url_name")
        self.max_items = kwargs.pop("max_items", 1) # The logic for the the clean and validate method method should be updated to work for max_items > 1
        super().__init__(*args, **kwargs)

    def clean(self, value):
        value = super().clean(value)
        return self.queryset.get(id=value)

    def validate(self, value):
        if not self.queryset.filter(id=value).exists():
            raise forms.ValidationError("Invalid choice")

    def widget_attrs(self, widget):
        attrs = super().widget_attrs(widget)
        attrs.update(
            {
                "data-autocomplete-url": reverse(self.url_name),
                "data-max-items": self.max_items,
            }
        )
        return attrs
```
<details>
<summary>If you use django-filter</summary>

```python
    class TomSelectFilter(filters.CharFilter):
        field_class = TomSelectField
    
        def filter(self, qs, value):
            if value:
                return qs.filter(**{f"{self.field_name}": value})
            return qs

    class SomethingFilter(filters.FilterSet):
        item = TomSelectFilter(
            field_name="item",
            label="Search item",
            url_name="items_autocomplete",
            queryset=Item.objects,
        )
```    

</details>

## Write View and register urls

```python

def items_autocomplete(request):
    queryset = Item.objects.all()
    query = request.GET.get("q")
    filter_ = models.Q(name__icontains=query) if query else models.Q()
    objects = [
        {
            "label": v.get("name"),
            "value": str(v.get("id")),
        }
        for v in queryset.filter(filter_).values("id", "name")
    ]
    return JsonResponse(data=objects, safe=False)

autocomplete_urls = [path("items-autocomplete/", items_autocomplete, name="items_autocomplete")]
```

<details>
<summary>For some fancy shenanigans</summary>

```python
autocomplete_urls = []

def register_autocomplete_view(view):
    func_name = view.__name__
    autocomplete_urls.append(
        path(f"{func_name.replace('_', '-')}/", login_required(view), name=func_name)
    )
    return view


@register_autocomplete_view # the path name will have the same name as the function, this setup can be useful if you need to write a lot of these
def items_autocomplete(request):
    queryset = Item.objects.all()
    query = request.GET.get("q")
    filter_ = models.Q(name__icontains=query) if query else models.Q()
    objects = [
        {
            "label": v.get("name"),
            "value": str(v.get("id")),
        }
        for v in queryset.filter(filter_).values("id", "name")
    ]
    return JsonResponse(data=objects, safe=False)
```
</details>

### Add url to your global url config

Keep the path name simple and add thems at the beginning
```python
# urls.py

urlpatterns = autocomplete_urls + [path("", home, name="home")]

```

## Use the field in a form

```python
class SomethingForm(forms.Form):
    item = TomSelectField(
        label="Search items",
        url_name="items_autocomplete",
        queryset=Item.objects,
    )
```
### Render the form

Nothing special to do here, render the form as usual

```html
{{form.render}}
```

## Javascript to create the tom select

```javascript
const createTomSelect = (element, url, maxItems) => {
    new TomSelect(element, {
        create: false,
        placeholder: 'search query',
        maxItems: maxItems,
        valueField: 'value',
        labelField: 'label',
        searchField: 'label',
        load: function (query, callback) {

            var fullUrl = `${url}?q=` + encodeURIComponent(query);
            fetch(fullUrl)
                .then(response => response.json())
                .then(json => {
                    callback(json);
                }).catch(() => {
                    callback();
                });

        },
        // custom rendering functions for options and items
        render: {
            option: function (item, escape) {

                return `<div class="d-flex justify-content-between">
                        <span>${item.label}</span>
                    </div>`;
            },
        },
    });

}

// create a tom select for every element with a data attribute data-autocomplete-url
const fields = document.querySelectorAll('[data-autocomplete-url]');

for (field of fields) {
  createTomSelect(field, field.dataset.autocompleteUrl, field.dataset.maxItems);
}

// remove the ugly wrapper around the field
const wrapper = document.querySelector('.ts-wrapper');

wrapper.classList.remove('form-control');

```
