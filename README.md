# django-autocomplete-tom-select-htmx

```python
class TomSelectField(forms.CharField):
    def __init__(self, *args, **kwargs):
        self.queryset = kwargs.pop("queryset")
        self.url_name = kwargs.pop("url_name")
        self.max_items = kwargs.pop("max_items", 1)
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


class TomSelectFilter(filters.CharFilter):
    field_class = TomSelectField

    def filter(self, qs, value):
        if value:
            return qs.filter(**{f"{self.field_name}": value})
        return qs


autocomplete_urls = []

def register_autocomplete_view(view):
    func_name = view.__name__
    autocomplete_urls.append(
        path(f"{func_name.replace('_', '-')}/", login_required(view), name=func_name)
    )
    return view


@register_autocomplete_view
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

const fields = document.querySelectorAll('[data-autocomplete-url]');

for (field of fields) {
  createTomSelect(field, field.dataset.autocompleteUrl, field.dataset.maxItems);
}

// remove the ugly wrapper around the field
const wrapper = document.querySelector('.ts-wrapper');

wrapper.classList.remove('form-control');

```
