# Strawberry GraphQL Django extension

This library provides a toolset for GraphQL schema generation from Django models.

## Installing the package

```
pip install strawberry_graphql_django
```

## Sample project

Your boss asks you for a Django model called `Fruit`, which has two attributes, name and color.

```python
# models.py
from django.db import models

class Fruit(models.Model):
    name = models.CharField(max_length=20)
    color = models.CharField(max_length=20)
```

Soon after that, your boss asks you to implement an API for that model so that everyone can access our great fruit database from all over the world.

The `Fruit` model has name and color attributes and we want to publish both of them. The GraphQL output type for our model is generated by using the `strawberry_django.type` decorator. Both fields are char fields so we will need to use the built-in Python `str` type in our API.

```python
# types.py
import strawberry_django
from . import models

@strawberry_django.type(models.Fruit)
class Fruit:
    name: str
    color: str
```

The last step is to generate the `Query` type and the `Schema`, which we can do using the core package `strawberry`.

```python
# schema.py
import strawberry
from typing import List
from .types import Fruit

@strawberry.type
class Query:
    fruits: List[Fruit] = strawberry.django.field()

schema = strawberry.Schema(query=Query)
```

Finally we add a `AsyncGraphQLView` view to our list of urls so that we can start making our first queries.

```python
# urls.py
from django.urls import include, path
from strawberry.django.views import AsyncGraphQLView
from .schema import schema

urlpatterns = [
    path('graphql', AsyncGraphQLView.as_view(schema=schema)),
]
```

After that, once the development server is running, you can read your fruits from the database through a GraphQL request.

```graphql
query {
  fruits {
    name
    color
  }
}
# -> fruits: [{ name: "strawberry", color: "red" }]
```

## Model Relations

Your boss wants the models to be more scalable. In particular, they think encoding `color` as a string is too limiting.
Let's create another model called `Color` and add a foreign key relation between the `Fruit` and `Color` models.

```python
# models.py
from django.db import models

class Fruit(models.Model):
    name = models.CharField(max_length=20)
    color = models.ForeignKey('Color', related_name='fruits', on_delete=models.CASCADE)

class Color(models.Model):
    name = models.CharField(max_length=20)
```

We also need to add a GraphQL Type for `Color` and modify the existing `Fruit` type to reflect our changes.
The `auto` field type is used for automatic type resolution. `strawberry_django` goes through all fields and resolves field types. It also generates resolvers for relation fields for you.

```python
# types.py
import strawberry_django
from strawberry import auto
from typing import List
from . import models

@strawberry_django.type(models.Fruit)
class Fruit:
    id: auto
    name: auto
    color: 'Color'

@strawberry_django.type(models.Color)
class Color:
    id: auto
    name: auto
    fruits: List[Fruit]
```

This generates the following schema:

```graphql
type Color {
  id: ID!
  name: String!
  fruits: [Fruit!]
}

type Fruit {
  id: ID!
  name: String!
  color: Color!
}

type Query {
  fruits: [Fruit!]!
}
```

Now you can start making queries and request all fruits and their colors from the database.

```graphql
query {
  fruits {
    name
    color {
      name
    }
  }
}
# -> fruits: [
#   { name: "strawberry", color: { name: "red" } },
#   { name: "raspberry", color: { name: "yellow" } }
# ]
```
