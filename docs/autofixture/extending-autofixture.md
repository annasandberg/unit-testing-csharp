AutoFixture supports three main categories of extension points
- Specimen builders
- Behaviors
- Customizations

## Specimen builders

Specimen builders are components that can be used to customize the creation of objects that AutoFixture can't handle with the default setup.

A specimen builder must implement the `ISpecimenBuilder` interface that looks like the following snippet

```csharp
public interface ISpecimenBuilder
{
    object Create(object request, ISpecimenContext context);
}
```

A request can be virtually of any type but most commonly they are reflection types like `Type`, `PropertyInfo`, `ParameterInfo` and so on. The AutoFixture kernel uses a [chain of responsibility](http://en.wikipedia.org/wiki/Chain-of-responsibility_pattern) to explore all the available builders and stop when a builder able to satisfy the request is met, directly or indirectly.

The context parameter represents the context in which the request is being handled. It's interface exposes only the `Resolve` method that is used to invoke AutoFixture's chain of responsibility to generate a value.

Here is an example of a specimen builder used to create instances of a specific type.

```csharp
public class SampleValueObject
{
    public string StringValue { get; set; }
    
    public int IntValue { get; set; }
}

public class SampleValueObjectSpecimenBuilder : ISpecimenBuilder
{
    public object Create(object request, ISpecimenContext context)
    {
        if (request is Type type && type == typeof(SampleValueObject))
        {		
            return new SampleValueObject
            {
                StringValue = context.Resolve(typeof(string)) as string,
                IntValue = 42
            };
        }
        
        return new NoSpecimen();
    }
}
```

Specimen builder are added to a given fixture via its Customizations collection.

```csharp
[Test]
public void Fixture_uses_specimen_builder_to_create_value()
{
    // ARRANGE
    var fixture = new Fixture();
    fixture.Customizations.Add(new SampleValueObjectSpecimenBuilder());

    // ACT
    var obj = fixture.Create<SampleValueObject>();

    // ASSERT
    Assert.That(obj.IntValue, Is.EqualTo(42));
}
```

## Behaviors

Together with specimen builders, behaviors are part of the graph used to represent the chain of responsibility used to create values.

Unlike the specimen builders, that can be seen as the leaves of the graph and therefore can only respond to a request, behaviors have access to both the request and the response and can act or modify them.

For example, the `TracingBehavior`, which is included in the AutoFixture core package, can be used to track the chain of calls used to serve a certain request.

```csharp
var fixture = new Fixture();
fixture.Behaviors.Add(new TracingBehavior());
var value = fixture.Create<SampleValueObject>();
```

When executing the snippet above, the following will be printed on the standard out

```
  Requested: AutoFixture.Kernel.SeededRequest
    Requested: SampleValueObject
      Requested: System.String StringValue
        Requested: AutoFixture.Kernel.SeededRequest
          Requested: System.String
          Created: d2f151c4-afa6-414c-8168-07b4de1ad5ae
        Created: StringValued2f151c4-afa6-414c-8168-07b4de1ad5ae
      Created: StringValued2f151c4-afa6-414c-8168-07b4de1ad5ae
      Requested: Int32 IntValue
        Requested: AutoFixture.Kernel.SeededRequest
          Requested: System.Int32
          Created: 195
        Created: 195
      Created: 195
    Created: SampleValueObject
  Created: SampleValueObject
```

Behaviors implement the `ISpecimenBuilderTransformation` interface. This is what the interface looks like

```csharp
public interface ISpecimenBuilderTransformation
{
    ISpecimenBuilderNode Transform(ISpecimenBuilder builder);
}
```

## Customizations

Finally, AutoFixture uses classes implementing the `ICustomization` interface to wrap in a single place multiple changes to the fixture configuration.

The `ICustomization` interface exposes a single method, `Customize`, that accepts a fixture and applies configuration changes to it.

```csharp
public interface ICustomization
{
    void Customize(IFixture fixture);
}
```

For example, the customization below injects values for `int` and `string`.

```csharp
public class TestCustomization : ICustomization
{
    public void Customize(IFixture fixture)
    {
        fixture.Customize<SampleValueObject>(o => o.With(p => p.StringValue, "FooBar"));

        fixture.Inject("Hello world");
        
        fixture.Inject(42);
    }
}
```

The customization can then be used as it follows.

```csharp
[Test]
public void TestCustomization_injects_constant_values()
{
    // ARRANGE
    var fixture = new Fixture();
    fixture.Customize(new TestCustomization());

    // ACT
    var testValue = fixture.Create<(int value, string message)>();

    // ASSERT
    Assert.That(testValue.value, Is.EqualTo(42));
    Assert.That(testValue.message, Is.EqualTo("Hello world"));
}

[Test]
public void TestCustomization_mixes_values()
{
    // ARRANGE
    var fixture = new Fixture();
    fixture.Customize(new TestCustomization());

    // ACT
    var testValue = fixture.Create<SampleValueObject>();

    // ASSERT
    Assert.That(testValue.value, Is.EqualTo(42));
    Assert.That(testValue.message, Is.EqualTo("FooBar"));
}
```

In the snippet above you can see how AutoFixture uses the injected values together with the ones specified in the customization when constructing an instance of `SampleValueObject`.