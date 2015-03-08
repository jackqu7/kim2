Serializers
-----------

*Serializers* are the building blocks of Kim - they define how JSON output should
look and how input JSON should be expected to look.

Serializers consist of *Fields*. Fields define the shape and nature of the data
both when being serialised and marshaled.

Serializers must define a `__model__`. This is the type that will be instantiated
if a new object is marshaled through the serializer. If you only want a simple
object, you can set this to `dict` or `object`.

.. code-block:: python

    class AuthorSerializer(Serializer):
    	__model__ = Author

      name = StringField()
      date_of_birth = DateField()

`AuthorSerializer` can now be used to serializer an Author object from the ORM:

.. code-block:: python

	>>> author = Author(name='JK Rowing', date_of_birth=date(1975, 3, 4))
	>>> AuthorSerializer(author).serialize()
	{'name': 'JK Rowling', 'date_of_birth': '1975-03-04'}


And the same Serializer can be used to marshal JSON data back into a new author
object.

.. code-block:: python

	>>> data = {'name': 'JK Rowling', 'date_of_birth': '1975-03-04'}
	>>> author = AuthorSerializer().marshal(data)
	<Author object>
	>> author.name
	'JK Rowling'

If an author object is already existing, it can be passed to the serializer
and it will be updated, as opposed to a new object being created.

.. code-block:: python

	>>> author = Author(name='JK Rowing', date_of_birth=date(2003, 6, 2))
	>>> data = {'name': 'JK Rowling', 'date_of_birth': '1975-03-04'}
	>>> author = AuthorSerializer(author).marshal(data)
	<Author object>
	>> author.date_of_birth
	date(1975, 3, 4)


Inputters and Outputters
------------------------

Fields can have multiple *outputters* attached
which control the way the field is represented in output and *inputters*
which are responsible for validating and converting JSON input back into
Python types.

Inputters and outputters are called one after the other in a chain. If any one
in the chain raises an exception, execution of the rest of the chain will not
take place and an exception returned to the calling code. This allows validation
to be implmented using inputters (and outputters if so desired.)

Commonly, you will use the same combination of outputters and
inputters for fields of the same type. For example a string type would require
the `output_str` outputter and the `input_str` inputter. For convinience, Kim
defines a number of *TypedFields* with these options already set, such as `StringField`.
(You can still add extra outputters and inputters to TypedFields).

Kim comes with a library of inputters for common validation tasks, such as `required`
and `length_between`.

.. code-block:: python

    class AuthorSerializer(Serializer):
    	__model__ = Author

        name = StringField(inputters=[required(), length_between(3, 20)])
        date_of_birth = DateField()

     >>> data = {date_of_birth': '1975-03-04'}
     >>> author = AuthorSerializer().marshal(data)
     Traceback (most recent call last):
         ...
     InputError: AuthorSerializer.name: is a required field

You can define your own inputters and outputters as callables which take
a data input, transform it if required and then return it. Alternatively they
can raise an Exception if they are not happy with the data passed.

.. code-block:: python

	def uppercase(field, data):
		"""Outputter to transform data to uppercase"""
		return data.upper()

	def must_equal(field, data):
		"""Inputter which will fail unless the data it recieves is equal to
		match"""
		if data != field.match:
			raise InputterError('does not match')
		return inputter

	class AuthorSerializer(Serializer):
		__model__ = Author

	    name = StringField(ExtraInputter(must_equal, before=set_source_field),
                         ExtraOutputter(uppercase, before=output_as_string),
                         match='Bob')
	    date_of_birth = DateField()

Note the kwarg `match` is passed to StringField and can be used in the outputter.
StringField will store any kwargs passed to it and they can be used by any
inputter/outputter, or multiple inputters/outputters.

Because inputters and outputters are passed the serializer as well as the data,
they can perform more complex tasks requiring the knowledge of multiple fields.
For example a composite field could be implemented as:

.. code-block:: python

	def full_name(field, data):
    serializer_data = field.serializer.data
		return serializer_data.first_name + ' ' + serializer_data.last_name

	class AuthorSerializer(Serializer):
		__model__ = Author

	    name = Field(outputters=[full_name(), output_as_string()])
	    date_of_birth = DateField()

Note the use of `Field` rather than `StringField` - this is because we are no
longer interested in the default source of `name` on the object, so we can't
use `StringField` any more as it expects it's source to be present on the object.
You could subclass `Field` to create a new `FullNameField` if so desired.

Nested Serializers
------------------
More complex output formats require child objects to be nested within parent
objects. These can be defined using *NestedField*.

.. code-block:: python

	class AuthorSerializer(Serializer):
		__model__ = Author

	    name = StringField()
	    date_of_birth = DateField()

	class BookSerializer(Serializer):
		__model__ = Book

	    title = StringField()
	    author = NestedField(AuthorSerializer)

  >>> author = Author(name='JK Rowing', date_of_birth=date(1975, 3, 4))
  >>> book = Book(title='Harry Potter', author=author)
  >>> BookSerializer(book).serialize()
  {'title': 'Harry Potter',
   'author': {'name': 'JK Rowling', 'date_of_birth': '1975-03-04'}}

  >>> data = {
        'title': 'Harry Potter',
        'author': {'name': 'JK Rowling', 'date_of_birth': '1975-03-04'}}
  >>> book = BookSerializer().marshal(data)
  >>> book.title
  'Harry Potter'
  >>> book.author.name
  'JK Rowling'


Collections
-----------
Where you require input/output be in the form of a list of scalar types or a list
of nested objects, use *Collections*.

Collections take an inner type and return a list of objects serialised/marshalled
into this type.

.. code-block:: python

  class BookSerializer(Serializer):
    __model__ = Book

    title = StringField()
    tags = CollectionField(StringField)


Roles
-----
Often you will want to only show a subset of fields on your serializer depending
on the circumstances. You can do this with roles. Kim also defines an implicit
`__default__` role which will be used if no other role is specified. By default
this contains all fields on the Serializer, but you can change that as well
if you wish.

Roles can be defined in terms of blacklists and whitelists, or in terms of
other roles using the + and - operators.

.. code-block:: python

    class AuthorSerializer(Serializer):
      __model__ = Author

      name = StringField()
      date_of_birth = DateField()
      address = StringField()
      postcode = StringField()
      country = StringField()

      __roles__ = {
        '__default__': whitelist('name', 'date_of_birth')
        'full': whitelist('name', 'date_of_birth', 'address', 'postcode', 'country')
        'with_address': role('__default__') + whitelist('address', 'postcode')
      }
