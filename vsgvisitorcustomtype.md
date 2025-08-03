# vsgvisitorcustomtype

## Overview

The `vsgvisitorcustomtype` example demonstrates how to create custom node types and visitors that can process them. It shows two different approaches for implementing visitor support for user-defined node classes.

## Key VSG Features Used

- **vsg::Inherit<>** - Template for creating custom node types
- **Custom visitors** - Extending visitor pattern for new types
- **Type casting** - Using cast<> for safe type conversion
- **Visitor dispatch** - Custom handling of derived types
- **Multiple inheritance approaches** - Two patterns for custom types

## What the Example Demonstrates

1. **Creating Custom Node Types**
   - Deriving from VSG node classes using vsg::Inherit
   - Adding custom properties to nodes
   - Proper destructor handling

2. **Custom Visitor Support - Approach 1**
   - Using a base visitor class with handleCustomGroups()
   - Manual type checking and dispatch
   - Virtual apply() methods for each custom type

3. **Custom Visitor Support - Approach 2**
   - Alternative implementation pattern
   - Different dispatch mechanism
   - Comparison of approaches

## Code Highlights

```cpp
// Define custom node types using vsg::Inherit
class CustomGroupNode : public vsg::Inherit<vsg::Group, CustomGroupNode>
{
public:
    std::string name = "car";
protected:
    ~CustomGroupNode() = default;
};

class CustomLODNode : public vsg::Inherit<vsg::Group, CustomLODNode>
{
public:
    double maxDistance = 1.0;
protected:
    ~CustomLODNode() = default;
};

// Custom visitor base class
class CustomVisitorBase : public vsg::Inherit<vsg::Visitor, CustomVisitorBase>
{
public:
    // Handle custom types in Group's apply method
    inline bool handleCustomGroups(vsg::Group& group)
    {
        if (auto cgn = group.cast<CustomGroupNode>()) {
            apply(*cgn);
            return true;
        }
        else if (auto cln = group.cast<CustomLODNode>()) {
            apply(*cln);
            return true;
        }
        return false;
    }

    // Virtual methods for custom types
    virtual void apply(CustomGroupNode& /*node*/) {}
    virtual void apply(CustomLODNode& /*node*/) {}
};

// Concrete visitor implementation
class VisitCustomTypes : public CustomVisitorBase
{
public:
    void apply(CustomGroupNode& node) override
    {
        std::cout << "CustomGroupNode: " << node.name << std::endl;
        node.traverse(*this);
    }
    
    void apply(CustomLODNode& node) override
    {
        std::cout << "CustomLODNode: " << node.maxDistance << std::endl;
        node.traverse(*this);
    }
};
```

## Running the Example

```bash
./vsgvisitorcustomtype
```

### Expected Output

```
First approach to implementing custom types and custom visitors that support these custom types.
apply(Group& node)
apply(CustomGroupNode& node) name = car
apply(CustomLODNode& node) maxDistance = 1

Second/Alternate approach to implementing custom types and custom visitors that support these custom types.
apply(Group& node)
apply(AlternateCustomGroupNode& node) name = car
apply(AlternateCustomLODNode& node) maxDistance = 1
```

## Key Takeaways

- VSG's visitor pattern can be extended for custom node types
- vsg::Inherit<> provides proper RTTI and reference counting for custom classes
- The cast<> method provides safe type conversion with nullptr on failure
- Custom visitors can handle both standard and custom node types
- Multiple dispatch patterns are possible depending on requirements
- Always use protected destructors for reference-counted objects
- Custom properties can be added to nodes for application-specific data