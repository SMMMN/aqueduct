# Validating Data

Data is added to a database through `update` and `insert` queries. As part of these two operations, a `ManagedObject<T>` will ensure that its properties have valid values. For example, a `Person` object might ensure that its name starts with a capital letter and that its phone number has only numeric values.  If one or more validation fails, the update or insert operation will fail and the data is not sent to the database. A validation failure will throw a `QueryException`, which automatically sends an HTTP response with error messaging to help the client correct their request.

The preferred way of setting a validation is to add `Validate` metadata to properties of a persistent type of a `ManagedObject<T>`. Here's an example of a validation that ensures a tweet is less than 140 characters:

```dart
class Tweet extends ManagedObject<_Tweet> implements _Tweet {}
class _Tweet {
  @managedPrimaryKey
  int id;

  @Validate.length(lessThan: 140)
  String message;
}
```

### Built-in Validators

There are a handful of built-in validations for common operations. For example, it is common to apply a regular expression to a value to ensure it formatted correctly or restrict the possible values to a list of available options. Common validators are available as named constructors on the `Validate` class. Here is an example:

```dart
class _Story {
  @managedPrimaryKey
  int id;

  @Validate.oneOf(const ["started", "accepted", "rejected", "delivered"])
  String state;
}
```

A built-in validator is useful because it automatically generates an error message that is returned in an HTTP response. For example, the previous code snippet indicates that the `state` property must be one of the four listed strings. If a value other than one of those four strings is used, the error message returned to the HTTP client would be:

```
"The value `invalidValue` is not valid for `state`. Valid values are: 'started', 'accepted', 'rejected', 'delivered'.".
```

See the API reference for `Validate` and its named constructors for possible options.

`Validate` metadata on transient properties have no effect. This metadata is only valid for database-backed properties declared in a persistent type.

### Custom Validators

There will be times where the built-in validators are not sufficient for your application's use case. You may create subclasses of `Validate` to provide custom validation behavior.For example, a `Validate` subclass you have declared named `ValidatePhoneNumber` would be used like so:

```dart
class _Person {
  @managedPrimaryKey
  int id;

  @ValidatePhoneNumber()
  String phoneNumber;
}
```

A subclass of `Validate` must override `Validate.validate()` and call the superclass' primary constructor when instantiated. Here's an example of that phone number validator:

```dart
class ValidatePhoneNumber extends Validate<String> {
  ValidatePhoneNumber({bool onUpdate: true, bool onInsert: true}) :
    super(onUpdate: onUpdate, onInsert: onInsert);

  @override
  bool validate(
      ValidateOperation operation,
      ManagedAttributeDescription property,
      String value,
      List<String> errors) {  
    if (value.length != 15) {
      errors.add(
        "${property.name} has invalid length of ${value.length}, must be 15 digits.");
      return false;
    }

    if (containsNonNumericValues(value)) {
      errors.add(
        "${property.name} has invalid format, must contain characters 0-9 only.");
      return false;
    }

    return true;
  }
}
```

Note that `Validate` is generic - you may provide a type argument which must match the type of the `value` argument to `Validate.validate()`. Omitting the type argument defaults to `dynamic` and therefore `value` must be `dynamic`.

The `validate` method must return `false` if validation failed, otherwise it should return `true`. It should add error messages for any failed validations. These error messages are returned in the HTTP response.

### Validation Behavior

A property may have more than one `Validate` metadata. In this case, all of the validations for a property must pass. The order in which multiple validations are performed is undefined and subject to change. Here's an example of validations that ensure a property's value is 10 characters long and only contains 10 alphabetic capital letters:

```dart
@Validate.length(equalTo: 10)
@Validate.matches(r"$[A-Z]+^")
String tenCapitalLetters;
```

By default, validations are executed when a `Query<T>`'s `insert` or `update` method is invoked. A validator can be restricted to only run on `insert` or `update` by passing values for its optional constructor arguments `onUpdate` and `onInsert`:

```dart
@Validate.matches(r"^[A-Z]+$", onInsert: true, onUpdate: false)
String validateOnInsertOnly;
```

It is important to understand how validations work when a value for a property is *not* specified in an insert or update query. For example, consider a `Person` with a `name` and `email` property and then inserted in a query that omits `email`:

```dart
var query = new Query<Person>()
  ..values.name = "Bob";

await query.insert();
```

Because `email` was not set on `Query.values`, validations will not be run on that property.

There are two special validators that can require a property to be set, or require that a property *not* be set. `Validate.present()` requires that the associated property must have a value. A property with this validator must be provided each time the object is inserted or updated. Like all validators, `Validate.present()` can be adjusted to be invoked on only inserts or only updates. For example, the following declaration requires that `email` is set on insertion, but doesn't have to be for updates:

```dart
@Validate.present(onUpdate: false, onInsert: true)
String email;
```

The inverse of `Validate.present()` is `Validate.absent()`. This validation prevents a property from being set. This is useful when a value should be included during insertion, but can't be updated. Here's an example:

```dart
@Validate.absent(onUpdate: true, onInsert: false)
String canOnlyBeSetOnce;
```

In the above declaration, the validator is only run on update operations and ensures that the property `canOnlyBeSetOnce` does not have a value. Because this validator is not run on insert operations, there is no restriction when the object is first inserted.

Validators are not run when a value is null. For example, the following insertion explicitly inserts `null` for the property `email`:

```dart
var query = new Query<Person>()
  ..values.email = null
  ..values.name = "Bob";

await query.insert();
```

Nullability is enforced by `ManagedColumnAttributes.isNullable` property. Consider the following declaration:

```dart
@ManagedColumnAttributes(nullable: false)
@Validate.length(greaterThan: 10)
String name;
```

Here, the property `name` must not be null and must be greater than 10 characters long. The behavior of inserting or updating this property is shown in the following table.

| Input Value for Name | Validation Runs? | Outcome |
|-|-|-|
| Insert value longer than 10 characters | Yes | Successful database insert |
| Insert value shorter than 10 characters | Yes | Database insert not executed, exception thrown |
| Insert value not specified | No | Database insert fails with non-null violation, exception thrown |
| Insert value is null | No | Database insert fails with non-null violation, exception thrown |
| Update value longer than 10 characters | Yes | Successful database update |
| Update value shorter than 10 characters | Yes | Database update not executed, exception thrown |
| Update value not specified | No | Successful database update |
| Update value is explicit null | No | Successful database update |

This behavior allows `ManagedObject<T>` instances to be partially updated, which also allows for partial PUTs. Partial PUTs can be prevented by adding `Validate.present()` metadata to all properties. While partial PUTs are not idiomatic REST, they are pragmatic software development.

This also means that any custom validator can safely assume that a value passed to `Validate.validate()` is non-null.

### Other Validator Behavior

For validators that can't be built by subclassing `Validate`, you may override `ManagedObject<T>.validate()`. This method is useful when a validation involves more than one property.

This method is passed the type of operation triggering the validation - either an insert or update. Here's an example:

```dart
class Person extends ManagedObject<_Person> implements _Person {
  @override
  bool validate(
      {ValidateOperation forOperation: ValidateOperation.insert,
      List<String> collectErrorsIn}) {
   var valid = super.validate(
     forOperation: forOperation, collectErrorsIn: collectErrorsIn);

    if (a + b > 10) {
      valid = false;
      collectErrorsIn.add("a + b must be greater than 10");
    }

    return valid;
  }
}
```

When overriding this method, the `super` implementation must be invoked to run validations for individual fields. Of course, if it returns `false`, the overridden method must return `false`.

### Skipping Validations

Validations are only run when values are set via `Query<T>.values`. Values set via `Query<T>.valueMap` are not validated. Therefore, objects should typically be inserted and updated using `Query<T>.values` unless validation must be ignored. Here's an example of skipping validation:

```dart
var query = new Query<Person>()
  ..valueMap = {
    "name" : "xyz",
    "email" : "whatever"
  };
```

Skipping validation should be rare.


### Update and Insert Callbacks

`ManagedObject<T>` subclasses may override `willUpdate` and `willInsert` to modify its properties prior to being updated or inserted by a `Query<T>`. For example, a managed object may have updated and created dates that can be guaranteed to be set when inserted or updated:

```dart
class Person extends ManagedObject<_Person> implements _Person {
  @override
  void willUpdate() {
    updatedAt = new DateTime.now().toUtc();
  }

  @override
  void willInsert() {
    createdAt = new DateTime.now().toUtc();
  }
}
class _Person {
  @managedPrimaryKey
  int id;

  String name;
  DateTime createdAt;
  DateTime updatedAt;
}
```

Note that all operations must be synchronous in these methods.

Both `willUpdate` and `willInsert` are invoked prior the validation phase. Thus, any values set in these methods will be subject to the declared validations of the instance.

Like validations, `willUpdate` and `willInsert`  are skipped when using `Query.valueMap`.