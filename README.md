# ChrisUnrealKit

A collection of useful, standalone tools and utilities for Unreal Engine 5 development.

## Tools

### EnhancedJson

A utility class that extends Unreal's native `FJsonObjectConverter` by adding hooks for custom post-processing. This gives you an opportunity to run your own logic after the standard `UPROPERTY` conversion, enabling flexible handling of dynamic fields and partial updates.

For detailed documentation and more examples, see the [EnhancedJson README](./Source/ChrisUnrealKit/EnhancedJson/README.md).

#### Core Feature: Post-Processing Hooks

The most important feature is the ability to add `PostJsonImport` and `PostJsonExport` methods to your `USTRUCT`. `EnhancedJsonConverter` will automatically call these functions, allowing you to serialize or deserialize fields that the standard converter would otherwise ignore.

This approach works well for dynamic data stored in a `TSharedPtr<FJsonObject>`.

```cpp
USTRUCT()
struct FSaveGameData
{
    GENERATED_BODY()

    UPROPERTY() 
    FString SaveSlotName;

    // A non-UPROPERTY field
    TSharedPtr<FJsonObject> DynamicPlayerData; 

    // This hook lets you populate 'DynamicPlayerData' from the source JSON object.
    void PostJsonImport(const TSharedPtr<FJsonObject>& JsonObject)
    {
        // Your custom logic to read from the JsonObject...
    }

    TSharedPtr<FJsonObject> PostJsonExport(TSharedPtr<FJsonObject> JsonObject) const
    {
        // Your custom logic to post process the JsonObject...
    }
};
```

#### Usage

Use `ChrisUE::FEnhancedJsonConverter` for a familiar API that incorporates this new flexibility. It includes functions for both full and partial updates, both of which respect your post-processing methods.

```cpp
// For full conversion (with post-processing)
ChrisUE::FEnhancedJsonConverter::JsonStringToUStruct(JsonString, MyStruct);

// For partial updates (with post-processing)
ChrisUE::FEnhancedJsonConverter::PartialJsonStringToUStruct(JsonString, MyStruct);
```
