Goals/Usecases
--------------
(in no particular order)

1. roles need to be seamless and not annoying
2. should be a default role
3. roles should extend to nested fields
4. roles should be combinable in a natural and flexible way

5. fields should know about the top level data
6. should have reusable validators that can be combined in a flexible way

7. should not have separate types for sqa stuff

8. serializers should be first class citizens
9. serializers should know about their parent and ancestors

10. should be simple to support a list of ids as opposed to a list of objects

11. should support property style functions

12. composite fields both when serializing and when marshalling

13. internals should be simple to understand - no impenetrable visitor patterns etc

Serializers
-----------

Serializers are themselves be responsible for serialising and marshaling,
rather than an additional Mapping class. (goal 8, goal 13)

.. code-block:: python

    # old way:
    AuthorSerializer().serialize(author)

    # new way:
    AuthorSerializer(author).serialize()

Serializers are now instantiated with the object they refer to, rather
than it being passed to serialize. This means fields can inspect and manipulate
it if required. Serializers and Fields are no longer stateless - they own their
own data and new instances are created each time a serialization/marshaling
is performed (goal 5, goal 8, goal 13)

Serializers have a `parent` attribute if they are being nested (set to None if not)
which allows you to access data from the parent, for example in getter functions.
It also makes it much simpler to implement the complex scenarios involved in
SQA getter functions without the need for complex solutions such as the visitor
pattern. (goal 7, goal 9, goal 13)


Input and Output pipelines do everything
----------------------------------------

The inputter and outputter pipelines make it simple to create reusable
validators and to implement complex fields like nested and collections. (goal 6, goal 11)

They are callables which return callables, which gives a lot of flexibility.
Objects implementing `__call__` could be used if required.

Serializers do very little themselves. The pipeline of inputters/outputters
are responsible for 99% of the process. The examples in the main doc are
syntactic sugar to make it easier to use, but what's really happening is this:

.. code-block:: python

    class AuthorSerializer(Serializer):
        __model__ = Author

        name = Field(outputters=[
            get_field_from_source,
            output_as_string
        ], inputters=[
            get_field_by_name,
            validate_is_string,
            set_source_field
        ])

So, the flow for outputting is, sequentially:

1. `get_field_from_source`: Retrieve the data. eg, `getattr(data, field.source)`
2. `output_as_string`: Passed the output from 1. and returns it as a string

For inputting:

1. `get_field_by_name`: Retrieve the data from the input, eg. `input[field.name]`
2. `validate_is_string`: Raise exception if data from 1. is not a string
3. `set_source_field`: Set the field on the resultant object, eg. `setattr(obj, field.source, data)`

This allows a huge amount of flexibility and means there is no additional
code required to handle Nested/Collection at the Serializer level, they simpler
require extra inputters/outputters. Even when SQA is involved everything can
be handled with such inputters/outputters.

Low level pipeline for nested:

.. code-block:: python
    class AuthorSerializer(Serializer):
        __model__ = Author

        [...]

    class BookSerializer(Serializer):
        __model__ = Book

        author = Field(outputters=[
            get_field_from_source,
            output_from_nested
        ], inputters=[
            get_field_by_name,
            validate_is_dict,
            input_from_nested
            set_source_field,
        ])

Low level pipeline for nested SQA (to marshal an author by ID):

.. code-block:: python
    class AuthorSerializer(Serializer):
        __model__ = Author

        [...]

    class BookSerializer(Serializer):
        __model__ = Book

        author = Field(outputters=[
            get_field_from_source,
            output_from_nested
        ], inputters=[
            get_field_by_name,
            validate_is_dict,
            extract_id,
            lookup_sqa_object_by_id,
            set_source_field,
        ])

Here `extract_id` would extract data['id'] and return it, this then gets passed
to lookup_sqa_object_by_id which returns the actual SQA object to be set on the
relationship.

This means if you wanted a foreign key field which works simply by an ID string
rather than a nested object it could be implemented like this: (goal 11)

.. code-block:: python

    class BookSerializer(Serializer):
        __model__ = Book

        author_id = Field(outputters=[
            get_field_from_source,
            output_as_string
        ], inputters=[
            get_field_by_name,
            validate_is_string,
            lookup_sqa_object_by_id,
            set_source_field,
        ])

This is identical except that the `extract_id` part has been removed. Various
combinations like this are possible to cater for different scenarios.

Because fields are entirely responsible for sourcing their own data, it becomes
possible to have composite fields which get/set data from multiple model
attributes, or indeed have no data at all (effectively a static field). (goal 12)


Limitations of this approach
----------------------------
The main problem here is that it's annoying to define all these inputters and
outputters the entire time. This is solved with syntactic sugar such as
`StringField` which includes them as defaults.

But this leads to a new problem - if you want to add your own
outputters/inputters you have to copy the entire chain again. Eg:

.. code-block:: python

    class AuthorSerializer(Serializer):
        __model__ = Author

        name = StringField(outputters=[
            get_field_from_source,
            MY_NEW_OUTPUTTER,
            output_as_string
        ])

Note it needs to go in the middle - not just be appended to the end.

This is not only annoying but will also cause problems if the expected ordering
changes in future versions.

Though this is a major limitation, there are two strategies in place to
mitigate it:

1. Outputters and inputters have access to the `Field` and any kwargs set on it.
This means that generic outputters and inputters can be defined which make use
of kwargs such as `source`. This should reduce the need for multiple combinations
of outputters/inputters to be used that often.

2. People will usually just want to shove something on immediately after they
get the data or immediately before it's outputted. Therefore you can pass
`ExtraInputter` and `ExtraOutputter` as args to a Field, which will insert it
in the right place. So the example above becomes:

.. code-block:: python

    class AuthorSerializer(Serializer):
        __model__ = Author

        name = StringField(ExtraOutputter(MY_NEW_OUTPUTTER, before=output_as_string))

As well as `before`, ExtraOutputter/ExtraInputter can also take `after` and
`replace`.

Application to "getter functions"
---------------------------------
Currently one of the most annoying and limited parts of Kim are getter functions
for NestedForeignKeys.

These have three main problems:

1. They can be reused, but only if you don't rely on self
2. If you do need self, you have to set it in the `__init__` method of the
Serializer, which is both annoying and makes it hard to understand how a
Serializer works at first glance.
3. It's impossible to get at data from other fields or from the parent field,
which is often required. (It can be worked around by passing things around
constantly, but it's a nightmare.)

Getter functions are now simply inputters and form part of the chain. They do
not need to be defined on serializers, but can still refer to attributes on the
serializer as they, like all inputters, are passed their `Field` which in turn
knows about it's parent `Serializer`. This means they can be reused without fuss.

Because they can access their Serializer, and Serializers now know about their
data, they can access other fields or even fields on the parent Serializer.

Example (using full syntactic sugar this time):

.. code-block:: python
    def author_getter(field, data):
        # data contains the ID we're after
        return db.session.query(Author) \
                         .filter(Author.type == field.serializer.data.type,
                                 Author.country == field.serializer.parent.data.country) \
                         .one()


    class AuthorSerializer(Serializer):
      __model__ = Author

      [...]


    class BookSerializer(Serializer):
        __model__ = Book

        type = StringField(choices=['fiction', 'non-fiction'])

        # Full syntactic sugar:
        author = NestedField(AuthorSerializer, getter=author_getter)
        # Which is equivilant to:
        author = NestedField(AuthorSerializer,
                             ExtraInputter(author_getter, replace=lookup_sqa_object_by_id)

Roles
-----
TBD, but as Serializers have full control over their own fields and data, and
roles belong to Serializers, it should be much easier to implement a fully
featured role systems.

Compatiability
--------------
Obviously this completely breaks the existing API. A compatiability layer
could be produced and would probably work but is unlikely to be worth it.

The best solution would be to install the old kim in a kim_legacy namespace
and keep using the old serializers, but define new ones going forward. A very
thin comptability layer could be placed on top of the old kim in order to maintain
the same API for the actual `serialize` and `marshal` functions, so views
would not need to care about which version they are using.
