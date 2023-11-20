# Response Model

Defining llm output schemas in Pydantic is done via `pydantic.BaseModel`. To learn more about models in pydantic checkout their [documentation](https://docs.pydantic.dev/latest/concepts/models/).

After defining a pydantic model, we can use it as as the `response_model` in your client `create` calls to openai. The job of the `response_model` is to define the schema and prompts for the language model and validate the response from the API and return a pydantic model instance.

## Prompting

When defining a response model, we can use docstrings and field annotations to define the prompt that will be used to generate the response.

```python
from pydantic import BaseModel, Field

class User(BaseModel):
    """
    This is the prompt that will be used to generate the response.
    Any instructions here will be passed to the language model.
    """
    name: str = Field(description="The name of the user.")
    age: int = Field(description="The age of the user.")
```

Here all docstrings, types, and field annotations will be used to generate the prompt. The prompt will be generated by the `create` method of the client and will be used to generate the response.

## Optional Values

If we use `Optional` and `default` they will be considered not required when sent to the language model

```python
class User(BaseModel):
    name: str = Field(description="The name of the user.")
    age: int = Field(description="The age of the user.")
    email: Optional[str] = Field(description="The email of the user.", default=None)
```

## Dynamic model creation

There are some occasions where it is desirable to create a model using runtime information to specify the fields. For this Pydantic provides the create_model function to allow models to be created on the fly:

```python
from pydantic import BaseModel, create_model


class FooModel(BaseModel):
    foo: str
    bar: int = 123


BarModel = create_model(
    'BarModel',
    apple=(str, 'russet'),
    banana=(str, 'yellow'),
    __base__=FooModel,
)
print(BarModel)
#> <class '__main__.BarModel'>
print(BarModel.model_fields.keys())
#> dict_keys(['foo', 'bar', 'apple', 'banana'])
```

??? notes "When would I use this?"

    Consider a situation where the model is dynamically defined, based on some configuration or database. For example, we could have a database table that stores the properties of a model for
    some model name or id. We could then query the database for the properties of the model and use that to create the model.

    ```sql
    SELECT property_name, property_type, description
    FROM prompt
    WHERE model_name = {model_name}
    ```

    We can then use this information to create the model.

    ```python
    types = {
        'string': str,
        'integer': int,
        'boolean': bool,
        'number': float,
        'List[str]': List[str],
    }

    BarModel = create_model(
        'User',
        **{
            property_name: (types[property_type], description)
            for property_name, property_type, description in cursor.fetchall()
        },
        __base__=BaseModel,
    )
    ```

    This would be useful when different users have different descriptions for the same model. We can use the same model but have different prompts for each user.

## Structural Pattern Matching

Pydantic supports structural pattern matching for models, as introduced by PEP 636 in Python 3.10.

```python
from pydantic import BaseModel


class Pet(BaseModel):
    name: str
    species: str


a = Pet(name='Bones', species='dog')

match a:
    # match `species` to 'dog', declare and initialize `dog_name`
    case Pet(species='dog', name=dog_name):
        print(f'{dog_name} is a dog')
        #> Bones is a dog
    # default case
    case _:
        print('No dog matched')
```

## Adding Behavior

We can add methods to our pydantic models just as any plain python class. We might want to do this to add some custom logic to our models.

```python
from pydantic import BaseModel
from typing import Literal

from openai import OpenAI

import instructor

client = instructor.patch(OpenAI())

class SearchQuery(BaseModel):
    query: str
    query_type: Literal["web", "image", "video"]

    def execute(self):
        # do some logic here
        return results


query = client.chat.completions.create(
        ..., response_model=SearchQuery
    )

results = query.execute()
```

Now we can call `execute` on our model instance after extracting it from a language model. If you want to see more examples of this checkout our post on [RAG is more than embeddings](../blog/posts/rag-and-beyond.md)