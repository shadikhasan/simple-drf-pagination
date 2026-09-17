# simple-drf-pagination

Zero-boilerplate pagination class factory for Django REST Framework (DRF).

`simple-drf-pagination` lets you configure page-number, limit/offset, or cursor
pagination inline—without writing a separate pagination subclass for every
view.

## Installation

```bash
pip install simple-drf-pagination
```

Requires Python 3.8 or later and Django REST Framework 3.14 or later.

## Quick start

Assign the class returned by `paginate()` to a DRF view or viewset:

```python
from rest_framework.viewsets import ModelViewSet
from simple_drf_pagination import paginate

from .models import Item
from .serializers import ItemSerializer


class ItemViewSet(ModelViewSet):
    queryset = Item.objects.all()
    serializer_class = ItemSerializer
    pagination_class = paginate(page_size=20, max_size=100)
```

Clients can then request a page and optionally choose its size:

```text
GET /items/?page=2&page_size=50
```

The default pagination type is page-number pagination, so
`paginate(...)` and `paginate("page", ...)` are equivalent.

## Pagination types

### Page number

```python
pagination_class = paginate("page", page_size=20, max_size=100)
```

```text
GET /items/?page=3&page_size=50
```

- `page_size` sets the default number of results per page.
- `max_size` limits the client-supplied `page_size` value.

### Limit and offset

```python
pagination_class = paginate("limit", page_size=50, max_size=200)
```

```text
GET /items/?limit=100&offset=200
```

- `page_size` becomes DRF's default limit.
- `max_size` limits the client-supplied `limit` value.

### Cursor

```python
pagination_class = paginate(
    "cursor",
    page_size=25,
    ordering="-created_at",
)
```

The first request does not need a cursor:

```text
GET /items/
```

Use the opaque cursor URL returned in the response to fetch the next or
previous page. The ordering field should be present on the queryset's model
and should provide a stable ordering. `max_size` does not apply to cursor
pagination.

## Response format

Page-number and limit/offset pagination responses include the total result
count, navigation links, and the serialized results:

```json
{
  "count": 42,
  "next": "http://localhost:8000/items/?page=2",
  "previous": null,
  "results": [
    {
      "id": 1,
      "name": "Item 1"
    }
  ]
}
```

Cursor pagination returns navigation links and results, but does not include a
total count:

```json
{
  "next": "http://localhost:8000/items/?cursor=encoded_cursor",
  "previous": null,
  "results": [
    {
      "id": 42,
      "name": "Item 42"
    }
  ]
}
```

The exact URLs and fields inside `results` depend on the request and serializer.

## API reference

```python
paginate(
    type: str = "page",
    page_size: int = 10,
    max_size: int = 100,
    ordering: str = "-id",
)
```

| Parameter | Description |
| --- | --- |
| `type` | Pagination strategy: `"page"`, `"limit"`, or `"cursor"`. |
| `page_size` | Default number of results returned per page. |
| `max_size` | Maximum client-requested size for page and limit pagination. |
| `ordering` | Ordering expression used by cursor pagination. |

The function returns a configured DRF pagination **class**, not an instance.
This is why it can be assigned directly to `pagination_class`.

Under the hood, each strategy subclasses the corresponding DRF class:

| Type | DRF base class | Configuration |
| --- | --- | --- |
| `"page"` | `PageNumberPagination` | `page_size`, `page_size_query_param="page_size"`, `max_page_size` |
| `"limit"` | `LimitOffsetPagination` | `default_limit`, `max_limit` |
| `"cursor"` | `CursorPagination` | `page_size`, `ordering` |

Passing any other type raises `ValueError`:

```python
paginate("unknown")  # ValueError: Invalid pagination type
```

## License

Released under the [MIT License](LICENSE).
