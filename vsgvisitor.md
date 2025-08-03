# vsgvisitor

## Overview

The `vsgvisitor` example demonstrates VSG's visitor pattern implementation, which is fundamental for traversing and processing scene graphs. It shows how to create custom visitors and use them to collect information about scene graph structure.

## Key VSG Features Used

- **vsg::Visitor/vsg::ConstVisitor** - Base classes for visitor pattern
- **accept/traverse** - Methods for visitor traversal
- **vsg::visit<>** - Template convenience function
- **Scene graph traversal** - Processing hierarchical node structures
- **Performance timing** - Measuring traversal performance

## What the Example Demonstrates

1. **Visitor Pattern Basics**
   - Creating custom visitors by inheriting from ConstVisitor
   - Overriding apply() methods for specific types
   - Collecting statistics during traversal

2. **Scene Graph Creation**
   - Building a quad tree structure
   - Recursive node creation
   - Testing traversal on large graphs

3. **Two Ways to Use Visitors**
   - Explicit visitor creation and accept() call
   - Using vsg::visit<> template for convenience

## Code Highlights

```cpp
// Create a custom visitor
struct MyVisitor : public vsg::ConstVisitor
{
    std::map<const char*, uint32_t> objectCounts;

    void apply(const vsg::Object& object) override
    {
        // Count objects by type
        ++objectCounts[object.className()];
        
        // Continue traversal to children
        object.traverse(*this);
    }
};

// Method 1: Explicit visitor usage
MyVisitor myVisitor;
scene->accept(myVisitor);
std::cout << "Found " << myVisitor.objectCounts.size() << " object types" << std::endl;

// Method 2: Using vsg::visit<> template
auto objectCounts = vsg::visit<MyVisitor>(scene).objectCounts;

// Build a quad tree for testing
vsg::ref_ptr<vsg::Node> createQuadTree(unsigned int numLevels)
{
    if (numLevels == 0) return vsg::Node::create();
    
    auto t = vsg::Group::create();
    t->children.reserve(4);
    
    --numLevels;
    for (int i = 0; i < 4; ++i) {
        t->addChild(createQuadTree(numLevels));
    }
    
    return t;
}
```

## Running the Example

```bash
# Default: 11 levels
./vsgvisitor

# Custom depth
./vsgvisitor --levels 5
./vsgvisitor -l 15
```

### Expected Output

```
VulkanSceneGraph Fixed Quad Tree creation : 0.0234
MyVisitor() object types s=2
    Group : 349525
    Node : 1398100

Using vsg::visit<>() object types s=2
    Group : 349525
    Node : 1398100
```

## Key Takeaways

- The visitor pattern is central to VSG's design for scene graph traversal
- Visitors can be const (read-only) or non-const (can modify the graph)
- The accept() method initiates traversal, traverse() continues to children
- Custom visitors override apply() methods for specific node types
- vsg::visit<> provides a convenient one-liner for simple traversals
- Visitor pattern enables type-safe processing without casting
- Performance is excellent even for large scene graphs (millions of nodes)