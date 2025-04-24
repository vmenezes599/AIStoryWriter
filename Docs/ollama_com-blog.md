# Structured outputs

## December 6, 2024

![Ollama playing with building blocks](https://ollama.com/public/blog/ollama-json.png)

Ollama now supports structured outputs making it possible to constrain a model’s output to a specific format defined by a JSON schema. The Ollama Python and JavaScript libraries have been updated to support structured outputs.

Use cases for structured outputs include:

- Parsing data from documents
- Extracting data from images
- Structuring all language model responses
- More reliability and consistency than JSON mode

### Get started

Download the latest version of [Ollama](https://ollama.com/download)

Upgrade to the latest version of the Ollama Python or JavaScript library:

Python

```bash
pip install -U ollama

```

JavaScript

```bash
npm i ollama

```

To pass structured outputs to the model, the `format` parameter can be used in the cURL request or the `format` parameter in the Python or JavaScript libraries.

#### cURL

```shell
curl -X POST http://localhost:11434/api/chat -H "Content-Type: application/json" -d '{
  "model": "llama3.1",
  "messages": [{"role": "user", "content": "Tell me about Canada."}],
  "stream": false,
  "format": {
    "type": "object",
    "properties": {
      "name": {
        "type": "string"
      },
      "capital": {
        "type": "string"
      },
      "languages": {
        "type": "array",
        "items": {
          "type": "string"
        }
      }
    },
    "required": [\
      "name",\
      "capital",\
      "languages"\
    ]
  }
}'

```

##### Output

The response is returned in the format defined by the JSON schema in the request.

```json
{
  "capital": "Ottawa",
  "languages": [\
    "English",\
    "French"\
  ],
  "name": "Canada"
}

```

#### Python

Using the [Ollama Python library](https://github.com/ollama/ollama-python), pass in the schema as a JSON object to the `format` parameter as either `dict` or use Pydantic (recommended) to serialize the schema using `model_json_schema()`.

```py
from ollama import chat
from pydantic import BaseModel

class Country(BaseModel):
  name: str
  capital: str
  languages: list[str]

response = chat(
  messages=[\
    {\
      'role': 'user',\
      'content': 'Tell me about Canada.',\
    }\
  ],
  model='llama3.1',
  format=Country.model_json_schema(),
)

country = Country.model_validate_json(response.message.content)
print(country)

```

##### Output

```py
name='Canada' capital='Ottawa' languages=['English', 'French']

```

#### JavaScript

Using the [Ollama JavaScript library](https://github.com/ollama/ollama-js), pass in the schema as a JSON object to the `format` parameter as either `object` or use Zod (recommended) to serialize the schema using `zodToJsonSchema()`.

```js
import ollama from 'ollama';
import { z } from 'zod';
import { zodToJsonSchema } from 'zod-to-json-schema';

const Country = z.object({
    name: z.string(),
    capital: z.string(),
    languages: z.array(z.string()),
});

const response = await ollama.chat({
    model: 'llama3.1',
    messages: [{ role: 'user', content: 'Tell me about Canada.' }],
    format: zodToJsonSchema(Country),
});

const country = Country.parse(JSON.parse(response.message.content));
console.log(country);

```

##### Output

```js
{
  name: "Canada",
  capital: "Ottawa",
  languages: [ "English", "French" ],
}

```

## Examples

### Data extraction

To extract structured data from text, define a schema to represent information. The model then extracts the information and returns the data in the defined schema as JSON:

```py
from ollama import chat
from pydantic import BaseModel

class Pet(BaseModel):
  name: str
  animal: str
  age: int
  color: str | None
  favorite_toy: str | None

class PetList(BaseModel):
  pets: list[Pet]

response = chat(
  messages=[\
    {\
      'role': 'user',\
      'content': '''\
        I have two pets.\
        A cat named Luna who is 5 years old and loves playing with yarn. She has grey fur.\
        I also have a 2 year old black cat named Loki who loves tennis balls.\
      ''',\
    }\
  ],
  model='llama3.1',
  format=PetList.model_json_schema(),
)

pets = PetList.model_validate_json(response.message.content)
print(pets)

```

#### Example output

```py
pets=[\
  Pet(name='Luna', animal='cat', age=5, color='grey', favorite_toy='yarn'),\
  Pet(name='Loki', animal='cat', age=2, color='black', favorite_toy='tennis balls')\
]

```

### Image description

Structured outputs can also be used with vision models. For example, the following code uses `llama3.2-vision` to describe the following image and returns a structured output:

![image](https://ollama.com/public/blog/beach.jpg)

```py
from ollama import chat
from pydantic import BaseModel

class Object(BaseModel):
  name: str
  confidence: float
  attributes: str

class ImageDescription(BaseModel):
  summary: str
  objects: List[Object]
  scene: str
  colors: List[str]
  time_of_day: Literal['Morning', 'Afternoon', 'Evening', 'Night']
  setting: Literal['Indoor', 'Outdoor', 'Unknown']
  text_content: Optional[str] = None

path = 'path/to/image.jpg'

response = chat(
  model='llama3.2-vision',
  format=ImageDescription.model_json_schema(),  # Pass in the schema for the response
  messages=[\
    {\
      'role': 'user',\
      'content': 'Analyze this image and describe what you see, including any objects, the scene, colors and any text you can detect.',\
      'images': [path],\
    },\
  ],
  options={'temperature': 0},  # Set temperature to 0 for more deterministic output
)

image_description = ImageDescription.model_validate_json(response.message.content)
print(image_description)

```

#### Example output

```py
summary='A palm tree on a sandy beach with blue water and sky.'
objects=[\
  Object(name='tree', confidence=0.9, attributes='palm tree'),\
  Object(name='beach', confidence=1.0, attributes='sand')\
],
scene='beach',
colors=['blue', 'green', 'white'],
time_of_day='Afternoon'
setting='Outdoor'
text_content=None

```

#### OpenAI compatibility

```py
from openai import OpenAI
import openai
from pydantic import BaseModel

client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")

class Pet(BaseModel):
    name: str
    animal: str
    age: int
    color: str | None
    favorite_toy: str | None

class PetList(BaseModel):
    pets: list[Pet]

try:
    completion = client.beta.chat.completions.parse(
        temperature=0,
        model="llama3.1:8b",
        messages=[\
            {"role": "user", "content": '''\
                I have two pets.\
                A cat named Luna who is 5 years old and loves playing with yarn. She has grey fur.\
                I also have a 2 year old black cat named Loki who loves tennis balls.\
            '''}\
        ],
        response_format=PetList,
    )

    pet_response = completion.choices[0].message
    if pet_response.parsed:
        print(pet_response.parsed)
    elif pet_response.refusal:
        print(pet_response.refusal)
except Exception as e:
    if type(e) == openai.LengthFinishReasonError:
        print("Too many tokens: ", e)
        pass
    else:
        print(e)
        pass

```

## Tips

For reliable use of structured outputs, consider to:

- Use Pydantic (Python) or Zod (JavaScript) to define the schema for the response
- Add “return as JSON” to the prompt to help the model understand the request
- Set the temperature to 0 for more deterministic output

## What’s next?

- Exposing logits for controlled generation
- Performance and accuracy improvements for structured outputs
- GPU acceleration for sampling
- Additional format support beyond JSON schema