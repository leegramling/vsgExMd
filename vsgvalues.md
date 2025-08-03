# vsgvalues

## Overview

The `vsgvalues` example demonstrates VSG's value wrapper system, which allows attaching typed values to objects as auxiliary data. It also shows how to create custom value types for user-defined structures.

## Key VSG Features Used

- **vsg::Value<T>** - Template class for wrapping values
- **setValue/getValue** - Object methods for attaching auxiliary data
- **Built-in value types** - intValue, floatValue, stringValue, etc.
- **Custom value types** - Creating value wrappers for user structs
- **Visitor pattern** - Traversing and processing different value types
- **Auxiliary data** - Extensible object metadata system

## What the Example Demonstrates

1. **Attaching Values to Objects**
   - Objects can store arbitrary typed values
   - Values are stored in the Auxiliary structure
   - Type-safe access to stored values

2. **Built-in Value Types**
   - `vsg::intValue` - Wraps int
   - `vsg::uintValue` - Wraps unsigned int
   - `vsg::floatValue` - Wraps float
   - `vsg::doubleValue` - Wraps double
   - `vsg::stringValue` - Wraps std::string

3. **Custom Value Types**
   - Creating value wrappers for user-defined structures
   - Type naming for serialization support
   - Direct access to wrapped values

## Code Highlights

```cpp
// Attach values to an object
object->setValue("name", "Name field contents");
object->setValue("time", 10.0);
object->setValue("size", 3.1f);
object->setValue("count", 5);

// Define a custom struct
namespace engine {
    struct property {
        float speed = 0.0f;
    };
    
    // Create a Value type for the struct
    using propertyValue = vsg::Value<property>;
}

// Register type name for serialization
EVSG_type_name(engine::propertyValue)

// Create and use custom value
auto my_property = engine::propertyValue::create(engine::property{10.0});
my_property->value().speed += 2.0f;  // Modify the wrapped value

// Visit different value types
struct VisitValues : public vsg::Visitor {
    void apply(vsg::intValue& value) override {
        std::cout << "intValue, value = " << value << std::endl;
    }
    void apply(vsg::stringValue& value) override {
        std::cout << "stringValue, value = " << value.value() << std::endl;
    }
};
```

## Running the Example

```bash
./vsgvalues
```

### Expected Output

```
Object has Auxiliary so check its ObjectMap for our values.
   key[count] intValue,  value = 5
   key[name] stringValue, value  = Name field contents
   key[pos] uintValue,  value = 4
   key[size] floatValue, value  = 3.1
   key[time] doubleValue, value  = 10

my_property = ref_ptr<engine::propertyValue>, className = engine::propertyValue
    after constructor my_property->value->speed = 10
    after increment my_property->value->speed = 12
    after multiplication my_property->value->speed = 24
```

## Key Takeaways

- VSG's value system provides type-safe metadata attachment to objects
- Values are stored only when needed (lazy allocation of Auxiliary)
- The visitor pattern enables type-specific processing of values
- Custom value types can wrap any user-defined structure
- Values are reference-counted like all VSG objects
- This system is used throughout VSG for extensible properties
- Type registration enables serialization support for custom values