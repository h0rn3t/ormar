
The use of ormar with fastapi is quite simple.

Apart from connecting to databases at startup everything else 
you need to do is substitute pydantic models with ormar models.

Here you can find a very simple sample application code.

!!!warning
    This example assumes that you already have a database created. If that is not the case please visit [database initialization][database initialization] section.


## Imports and initialization 

First take care of the imports and initialization 
```python hl_lines="1-12"
--8<-- "../docs_src/fastapi/docs001.py"
```

## Database connection 

Next define startup and shutdown events (or use middleware)
- note that this is `databases` specific setting not the ormar one
```python hl_lines="15-26"
--8<-- "../docs_src/fastapi/docs001.py"
```

!!!info
    You can read more on connecting to databases in [fastapi][fastapi] documentation

## Models definition 

Define ormar models with appropriate fields. 

Those models will be used insted of pydantic ones.
```python hl_lines="29-47"
--8<-- "../docs_src/fastapi/docs001.py"
```

!!!tip
    You can read more on defining `Models` in [models][models] section.

## Fastapi endpoints definition

Define your desired endpoints, note how `ormar` models are used both 
as `response_model` and as a requests parameters.

```python hl_lines="50-79"
--8<-- "../docs_src/fastapi/docs001.py"
```

!!!note
    Note how ormar `Model` methods like save() are available straight out of the box after fastapi initializes it for you.

!!!note
    Note that you can return a `Model` (or list of `Models`) directly - fastapi will jsonize it for you

## Test the application

### Run fastapi

If you want to run this script and play with fastapi swagger install uvicorn first

`pip install uvicorn`

And launch the fastapi.

`uvicorn <filename_without_extension>:app --reload`

Now you can navigate to your browser (by default fastapi address is `127.0.0.1:8000/docs`) and play with the api.

!!!info
    You can read more about running fastapi in [fastapi][fastapi] docs. 

### Test with pytest

Here you have a sample test that will prove that everything works as intended.

Be sure to create the tables first. If you are using pytest you can use a fixture.

```python
@pytest.fixture(autouse=True, scope="module")
def create_test_database():
    engine = sqlalchemy.create_engine(DATABASE_URL)
    metadata.create_all(engine)
    yield
    metadata.drop_all(engine)
```

```python

# here is a sample test to check the working of the ormar with fastapi

from starlette.testclient import TestClient

def test_all_endpoints():
    # note that TestClient is only sync, don't use asyns here
    client = TestClient(app)
    # note that you need to connect to database manually
    # or use client as contextmanager during tests
    with client as client:
        response = client.post("/categories/", json={"name": "test cat"})
        category = response.json()
        response = client.post(
            "/items/", json={"name": "test", "id": 1, "category": category}
        )
        item = Item(**response.json())
        assert item.pk is not None

        response = client.get("/items/")
        items = [Item(**item) for item in response.json()]
        assert items[0] == item

        item.name = "New name"
        response = client.put(f"/items/{item.pk}", json=item.dict())
        assert response.json() == item.dict()

        response = client.get("/items/")
        items = [Item(**item) for item in response.json()]
        assert items[0].name == "New name"

        response = client.delete(f"/items/{item.pk}", json=item.dict())
        assert response.json().get("deleted_rows", "__UNDEFINED__") != "__UNDEFINED__"
        response = client.get("/items/")
        items = response.json()
        assert len(items) == 0
```

!!!tip
    If you want to see more test cases and how to test ormar/fastapi see [tests][tests] directory in the github repo

!!!info
    You can read more on testing fastapi in [fastapi][fastapi] docs. 

[fastapi]: https://fastapi.tiangolo.com/
[models]: ./models.md
[database initialization]:  ../models/#database-initialization-migrations
[tests]: https://github.com/collerek/ormar/tree/master/tests