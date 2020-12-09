---
title: "Railway-Oriented Programming in C#"
date: "2020-10-09"
authors:
- github: mgw854
  name: "Matthew White"
categories: ["C#", "functional programming"]
featuredImage: "/images/railway-oriented-programming/featured.jpg"
---

At Plex, we're constantly looking for ways to improve and make our products better for our customers, developers, and everyone else who interacts with them. Taking inspriation from functional programming, and in particular, Rust, we have begun to adopt a more functional approach in certain key areas that reduces bugs and makes the code easier to follow, all while staying in C#, the primary language for Plex development.

## Work Smarter, Not Harder
C# is a great, general-purpose programming language. Since the introduction of LINQ in C# 3, this object-oriented language has taken on more functional aspects while still retaining a familiar feel to developers without a background in other languages like F# or Haskell. Even today, though, an imperative approach often feels more comfortable. For instance, imagine validating a `Person` DTO:

```csharp
public bool Validate(Person person)
{
  if (person == null) 
  { 
    throw new ArgumentNullException(nameof(person)); 
  }

  if (string.IsNullOrWhiteSpace(person.FirstName))
  {
    return false;
  }

  if (string.IsNullOrWhiteSpace(person.LastName))
  {
    return false;
  }

  if (person.Birthday > DateTime.Now)
  {
    return false;
  }

  return true;
}
```

In this overly-simplistic example, we have to consider not only our domain requirements (does the person have two valid names and a birthday in the past), we also have to consider the limitations of the language. `Person`, being a class, could be `null`, even if it doesn't make sense to call a method against a null parameter. Although nullable reference types in C# 8 make this particular situation better, they are not a panacea.

Building logic in an imperative manner like this can be intuitive for a few conditions, but quickly becomes unwieldy as the validation becomes more complex. Making a simple mistake, like accidentally returning or _not_ returning, can introduce serious issues such as the [goto fail](https://www.imperialviolet.org/2014/02/22/applebug.html) bug. Relying on humans to write perfect code every time, and other humans doing code reviews to catch these bugs, is error-prone. There has to be a better way.

## Rust & Monads

[Rust](https://www.rust-lang.org/) is a modern systems programming language that refutes the idea that a language cannot have both rigorous safety and high-level abstractions built-in. The Rust community calls these abstractions "zero-cost" because the compiler can rewrite the high-level code into assembly as efficient as if it had been hand-coded for performance. Because of these abstractions, Rust delivers developers into the [_pit of success_](https://blog.codinghorror.com/falling-into-the-pit-of-success/) by rejecting `null` and exceptions. Instead, Rust offers the `Option` and `Result` monads.

Monads are a functional programming concept that are often explained in terms comprehensible only by an academic computer scientist. Instead, it is helpful to think of monads as containers for a value that work with other containers. For instance, a `List<T>` is a monad that may contain nothing, one value, or many values. Using a _map_ operation (`Select` in C# vernacular), we can transform one `List<T>` into another.

### Option
As an alternative to `null`, Rust makes use of the `Option<T>` monad, which could contain nothing, or contain `Some(T)`. Importantly, a developer never has the chance to assume the state of the monad. The developer must always handle both the chance that the monad has no value and that it does.

```rust
let first_name = match person.first_name {
  Some(name) => name,
  None => "<Unknown>"
};
```

Notice the pattern-matching construct that allows us to "open up" the `Option<T>`, but only gives us access to the value if we've confirmed that there is in fact one. There is no way to accidentally dereference the value if it doesn't exist (eliminating the chance we encounter [Tony Hoare's billion-dollar mistake](https://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare/), the null-reference exception). We can also use built-in operations (like `map`) to work with monads easily without giving up their benefits. For instance, the following call only performs the mapping operation if `person.last_name` has `Some` value. If it is `None`, that is instead returned.

```rust
let last_initial = person.last_name.map(|x| x[0..1].to_string());
```

### Result
In terms of exceptions programmers consider, it's useful to bifurcate the set into _expected_ and _unexpected_ exceptions in our programs. Users typing in bad data is hardly an exceptional case; it is much more the usual case. There are any number of I/O operations that are also likely to fail, but only very rarely. Finally, there are those edge cases we never really consider, like a call to allocate memory failing. In C#, a `System.Exception` represents all these scenarios, no matter how unexceptional they may be. [Microsoft warns against using exceptions for control flow](https://docs.microsoft.com/en-us/visualstudio/profiling/da0007-avoid-using-exceptions-for-control-flow?view=vs-2017), but without other ready tools, developers often use them for that purpose. In Rust, the delineation is quite clear: if a developer expects failure is possible, use the `Result` monad; if the failure is unexpected or unrecoverable, use the `panic!` macro to crash the program (after all, it is better to crash than corrupt data).

The `Result` monad is a specialization of the more general-purpose `Either` monad. A `Result` can _either_ have a value (called `Ok`) or it may contain an error (called `Err`). Just as with the `Option` type, Rust forces developers to consider both cases whenever working with a `Result`:

```rust
use chrono::prelude::*;

pub enum ValidationError {
  NotAfterError { value: NaiveDate, max_value: NaiveDate },
  EmptyString
}

fn validate_birthday(birthday: NaiveDate) -> Result<NaiveDate, ValidationError> {
  let today = Local::now().date().naive_local();

  if today < birthday {
    return Err(ValidationError::NotAfterError { value: birthday, max_value: today });
  }

  Ok(birthday)
}

fn validate_name(name: &str) -> Result<&str, ValidationError> {
  let trimmed = name.trim();

  if trimmed.len() == 0 {
    return Err(ValidationError::EmptyString);
  }

  Ok(trimmed)
}

pub fn validate(person: Person) -> Result<Person, ValidationError> {
  validate_name(person.first_name)
  .and_then(|_| validate_name(person.last_name))
  .and_then(|_| validate_birthday(person.birthday))
}
```

In this example, we can condense all the validation (the `validate` function) into worrying about the happy path, and allowing the language semantics (through the use of the `Result` type and combinators) to handle the failure case for us. In a more forward-thinking example, our validation could even do data normalization, such as cleaning up whitespace, and pass this along to the next step. The only possible output of our validation in that case is the normalized, valid input _or_ a validation error. This pattern makes it very hard to do the wrong thing and access invalid input or ignore a fault.

## Applying to C#
There already exist libraries in C# for these monads, but in our review of them, they often allowed developers to take an easy escape hatch, like providing a property on an `Option` that would return the encapsulated value or throw an exception. While there are certainly cases where a developer knows more than the compiler and can make that decision, we thought it would be too easy for developers to slip into bad habits with such a new approach. Instead, we built a small library containing these monads without the escape hatches that we saw elsewhere.

Our implementation of the `Option` monad, called `Optional<T>` to mirror the pre-existing `Nullable<T>` (of which it shares many similarities), looks very simple at its core:

```csharp
public readonly struct Optional<T> : IEquatable<Optional<T>>, IEquatable<T>
{
  private readonly T _value;

  public Optional(T value)
  {
    _value = value;
    this.HasValue = true;
  }

  public bool HasValue { get; }

  public T GetValueOrDefault(T @default) => this.HasValue ? _value : @default;

  public bool Contains(T value)
  {
    return this.HasValue && (value?.Equals(_value) ?? _value == null);
  }

  public static implicit operator Optional<T>(T value) => new Optional<T>(value);
}
```

To make using `Optional<T>` easy-to-use, we provide an implicit conversion between `T` and `Optional<T>` to reduce code in the usual case. While it is trivial to check if the `Optional<T>` has a value, getting directly at that value requires providing a default if it does not exist. Of course, there are combinators like _map_ (called `Select` in C#) defined directly on the struct and as extension methods that allow for more complex scenarios, like manipulating a value only if it exists:

```csharp
public Optional<U> Select<U>(Func<T, U> map)
{
  return this.HasValue ? Optional.Some(map(_value)) : Optional.None;
}
```

The `Result<T, E>` monad implementation looks very similar when stripped down to its basic parts:

```csharp
public readonly struct Result<T, E> where E : Error
{
  private readonly T _value;
  private readonly E _error;

  internal Result(T value)
  {
    _value = value;
    _error = null;
  }

  internal Result(E error)
  {
    _error = error;
    _value = default;
  }

  /// <summary>
  /// Indicates whether or not the result is one of failure
  /// </summary>
  public bool IsError => _error != null;

  /// <summary>
  /// Indicates whether or not the result is one of success
  /// </summary>
  public bool IsOk => _error == null;
}
```

The `Result` monad is made useful through built-in helpers like `Then` (for only performing operations on an `Ok` value), `Handle` (for dealing only with errors), and `Select` (for handling both cases). These helpers also have asynchronous counterparts to enable more complex logic that might require external interaction.

```csharp
public Result<V, E> Then<V>(Func<T, Result<V, E>> fn)
{
  return this.IsOk ? fn(_value) : new Result<V, E>(_error);
}
    
public V Select<V>(Func<T, V> ok, Func<E, V> error)
{
  return this.IsOk ? ok(_value) : error(_error);
}

public Result<T, E> Handle(Func<E, Result<T, E>> fn)
{
  return this.IsError ? fn(_error) : this;
}
```

## Enhancing Validation Logic
Our forthcoming Plex Translation Management System makes heavy use of these monadic types. In particular, all validation logic is handled with `Result`s shuttling values back-and-forth, doing both error checking and data normalization. The Translation Management System is built in a domain-driven, CQRS fashion, with all data manipulation being handled by commands. We can easily tack on validation to each command, modifying it as it flows through the pipeline. Take, for example, the very simple to command to add support for a new locale:

```csharp
public sealed class IntroduceLocale : Command
{
  public string LanguageIdentifier { get; set; }
  public string ScriptIdentifier { get; set; }
  public string TerritoryIdentifier { get; set; }
  public bool Internal { get; set; }
}
```

These properties are user-defined inputs, and although they are at least constrained to the correct data types, we can make no other assumptions about them. The strings may be `null`, empty, padded with whitespace, contain invalid characters, or even, by chance, be valid. Each command passes through a validation handler, which is split into two parts: command invariant validation and aggregate invariant validation.

Going back to the principles of domain-driven design, any validation that can be done without knowing the state of the system is an _invariant_ of the command. These are rules about the business that cannot be violated. Written out for this command, they are easily understandable, but slightly complicated:
- `LanguageIdentifier` must be two or three lowercase letters (a-z) and be part of the [ISO 639-1 or ISO 639-2 specification](https://www.iso.org/iso-639-language-codes.html)
- `ScriptIdentifier` (optional) must be four letters (a-z), the first uppercase, the remainder lowercase and be part of the [Common Locale Data Repository](http://cldr.unicode.org/)'s script list
- `TerritoryIdentifer` (optional) must be two uppercase letters (A-Z), starting with an `X` or part of the [ISO-3166 alpha-2](https://www.iso.org/iso-3166-country-codes.html) code, or be three digits (0-9) and part of the Common Locale Data Repository's territory list

We can represent one set of invariants (for `TerritoryIdentifier`) like so, as an extension method on the command itself:

```csharp
public static Result<T, Error> ValidateTerritoryIdentifier<T>(this T @this, CldrRepository cldrRepo, string parameter, string name)
  where T : Command
{
  // Territory isn't required, so ignore any blank or empty values
  if (string.IsNullOrWhiteSpace(parameter))
  {
    return Result.Ok(@this);
  }

  // Two letter country codes conforming to ISO-3166 alpha-2, like "US" for the United States
  if (parameter.Length == 2 && parameter.All(c => c >= 'A' && c <= 'Z'))
  {
    // Country codes starting with X are reserved for private use
    if (parameter[0] == 'X' || cldrRepo.GetTerritoryCodes().Contains(parameter))
    {
      return Result.Ok(@this);
    }
  }
  // Three digit CLDR territory codes, like "419" to represent the region of Latin America
  else if (parameter.Length == 3 && parameter.All(c => c >= '0' && c <= '9') && cldrRepo.GetTerritoryCodes().Contains(parameter))
  {
    return Result.Ok(@this);
  }

  return Result.Error<T, Error>(new ValidationError(name,
    "The territory identifier must be an ISO 3166-1 alpha-2 code or otherwise conform to the Unicode territory subtag format.",
    parameter,
    LocalizeMessages.GetTerritoryIdentifierMustISO31661alpha2OrUnicode()));
}
```

Although that same `if`/`then` pattern repeats here in an almost imperative way, the total scope of the function is small and easily understood and audited (and, as a rule, all validation logic in the Plex Translation Management System is covered by tests over every line and branch). We also fail safe: unless we explicitly return an `Ok` value, an `Error`--a `ValidationError`, to be precise--is returned. This very defensive pattern helps keep bad data out of the system, and doing so reduces the potential for bugs to crop up.

Composed together, the validations form a chain that only needs to consider the happy path:
```csharp
private Result<IntroduceLocale, Error> ValidateCommandInvariants(IntroduceLocale command)
{
  return
    command
      .ValidateLanguageIdentifier(_cldrRepo, command.LanguageIdentifier, nameof(command.LanguageIdentifier))
      .Then(cmd => cmd.ValidateScriptIdentifier(_cldrRepo, cmd.ScriptIdentifier, nameof(cmd.ScriptIdentifier)))
      .Then(cmd => cmd.ValidateTerritoryIdentifier(_cldrRepo, cmd.TerritoryIdentifier, nameof(cmd.TerritoryIdentifier)));
}
```

Each validation takes as its input the command itself, and returns a command if it is valid (this command may be the same command as was passed to it, or a new one that has slightly different data as part of normalization). If any step fails, `Then` immediately returns the prior error without doing additional processing work.

Finally, the _aggregate_, the core of the business logic, needs to be validated based on the requested change. This may require getting values out of a database and comparing them against the command, which is best done asynchronously. Therefore, the aggregate invariants return a `Task<Result<T, E>>`, which works just as well as the bare `Result<T, E>`:

```csharp
public async Task<Result<IntroduceLocale, Error>> ValidateAggregateInvariantsAsync(IntroduceLocale command)
{
  // The only aggregate validation is if the locale already exists
  if ((await _repo.GetLocaleAsync(command.GetEntityId())).HasValue)
  {
    return Result.Error<IntroduceLocale, Error>(new PreexistingEntityError(command.GetEntityId()));
  }

  return Result.Ok(command);
}
```

Here, we make use of both monads. Rather than potentially return `null`, the repository (`_repo.GetLocaleAsync`) returns an `Optional<Locale>`--after all, database queries are not guaranteed to return results. We don't actually care about the returned value, however, so checking `HasValue` is sufficient to prove whether we have a pre-existing locale of the same definition. If so, an error (this time of type `PreexistingEntityError`) is returned.

These two chunks compose together easily, even though one is synchronous and one is asynchronous:

```csharp
public Task<Result<IntroduceLocale, Error>> HandleAsync(IntroduceLocale command)
{
  return this.ValidateCommandInvariants(command)
    .ThenAsync(this.ValidateAggregateInvariantsAsync);
}
```

## Bridging the Divide
Ultimately, our use of these new data types may make our validation code better, but is of no use if we can't safely bridge the divide between our boundaries. In the case of the Plex Translation Management System, using a hexagonal architecture, the boundary is between the ASP.NET Core application that hosts the API and the core business logic contained in a library project.

In the most simplistic case (querying for data), the return type from the core business logic is an `Optional<T>`. Using a simple `Case` combinator, a developer can translate that into an `ActionResult` usable by ASP.NET Core:

```csharp
public async Task<ActionResult<LocaleResponseModel>> GetLocale(string locale)
{
  return (await _localeRepo.GetLocale(locale))
    .Case<ActionResult<LocaleResponseModel>>(m => this.Ok(m), () => this.NotFound(locale));
}
```
If the requested locale exists, the model gets returned as the body of a `200 OK` response. If the requested locale does not exist, a `404 Not Found` response is sent to the client.

A more complex example involves the kind of validation we looked at earlier. Based on the kind of error that was returned on the `Result`, a different status code may be necessary. Luckily, we can build a helper method that translates a `Result` into the correct kind of `ActionResult`:

```csharp
public static ActionResult<T> AsResponseFor<T>(this Result<T, Error> result, string actionName, string controllerName, Func<T, object> routeValues)
{
  return result.Select(
    t => new AcceptedAtActionResult(actionName, controllerName.Replace("Controller", string.Empty), routeValues(t), null),
    ActionResultHelpers.CreateErrorActionResult<T>);
}

private static ActionResult<T> CreateErrorActionResult<T>(Error e)
{
  switch (e)
  {
    case EntityNotFoundError _:
      return new NotFoundObjectResult(e);
    case VersionConflictError _:
      return new ConflictObjectResult(e);
    case NoTenantSecurityAccessError _:
    case NoEnterpriseSecurityAccessError _:
      return new ObjectResult(e)
      {
        StatusCode = StatusCodes.Status403Forbidden
      };
    default:
      return new BadRequestObjectResult(e);
  }
}
```

The controller for supporting a new locale becomes essentially boilerplate:

```csharp
public Task<ActionResult<IntroduceLocale>> PostLocale([FromBody] IntroduceLocaleModel introduceLocaleModel)
{
  IntroduceLocale command = /* convert model to command */;
  return _introduceLocaleValidationHandler
    .HandleAsync(command)
    .PublishIfValidAsync(_publisher)
    .AsResponseForAsync(
      nameof(this.PostLocale),
      nameof(LocalesController),
      l => new
      {
        locale = new Locale(l.LanguageIdentifier, l.ScriptIdentifier, l.TerritoryIdentifier, LocaleStatus.New).ToString()
      });
}
```

## Concluding Thoughts
Railway-oriented programming reduces the number of local decisions and therefore the complexity of code. Less-complex code is easier for a developer to write, but more importantly, is easier to maintain over time--a cost we often discount. Types like `Optional<T>` and `Result<T, E>` can abstract away perilous decision-making processes for a developer, reducing the chance of or entirely eliminating certain classes of bugs and security vulnerabilities. Their ease of composition makes falling into the _pit of success_ easy, and with a little practice, converting `null`-and-exception-ridden code into something more functional using these types becomes second nature.

Although C# still retains its roots as an object-oriented programming language, the addition of tools like LINQ over the years make introducing concepts like monads and railway-oriented programming feel less alien in a C# code base. Rather that choose vocabularly that might align more with Haskell or the mathematical foundations of functional programming, we chose to retain the .NET vernacular (`Select` instead of `Map`, `Where` instead of `Filter`, etc.) in order to lower the barrier for adoption. Our Plex Translation Management System makes extensive use of these concepts throughout its code base, and is easier to test and maintain for it. Other applications in the Plex ecosystem have begun adopting these types as well, reducing errors casued by `null` values bubbling up or trying to catch the right exceptions at the right point in time. We will continue to explore railway-oriented programming and look at other benefits we can glean from the functional programming community.