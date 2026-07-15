---
name: class-structure
description: This skill defines the basic structure of a class in nopCommerce.
---
# Basic Structure of a class in nopCommerce

## When to use
Activate this skill when writing a Controller, Service, ModelFactory, Validator, Builder or any type of class. Use appropriate region block where appicable.

## How to use
- Always use the appropriate region block for the code being written. For example, if writing a method, it should be placed within the Methods region block.
- Private methods should be placed in the Utilities region block.

## Template
```csharp
public class {ClassName}
{
    #region Fields
    #endregion

    #region Properties
    #endregion

    #region Ctor
    #endregion

    #region Utilities
    #endregion

    #region Methods
    #endregion
}
```

## Guardrails
- Never use any other region block other than the ones defined in the template.
