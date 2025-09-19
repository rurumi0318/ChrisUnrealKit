# Enhanced JSON Converter for Unreal Engine 5

A simple extension to Unreal's built-in JSON conversion that supports dynamic fields and partial updates through optional post-processing methods.

## The Problem

Unreal's `FJsonObjectConverter` only handles `UPROPERTY` fields. This becomes a limitation when your JSON contains data that can't be represented as a `UPROPERTY`, such as a `TSharedPtr<FJsonObject>` for dynamic or unstructured data.

```cpp
USTRUCT()
struct FMyData
{
    GENERATED_BODY()

    UPROPERTY()
    FString Name;  // ✅ Works fine with standard conversion.

    TSharedPtr<FJsonObject> DynamicData;  // ❌ Can't be a UPROPERTY, so it's ignored by FJsonObjectConverter.
};
```

## The Solution: Post-Processing Methods

This converter extends the standard functionality by automatically calling optional `PostJsonImport` and `PostJsonExport` methods on your struct after the standard conversion is complete.

Simply add these methods to your struct to handle any custom logic.

```cpp
USTRUCT()
struct FMyData
{
    GENERATED_BODY()

    UPROPERTY()
    FString Name;

    TSharedPtr<FJsonObject> DynamicData;  // Your dynamic field

    // Called after standard import. Use it to populate non-UPROPERTY fields.
    void PostJsonImport(const TSharedPtr<FJsonObject>& JsonObject)
    {
        if (JsonObject.IsValid())
        {
            const TSharedPtr<FJsonObject>* DynamicPtr;
            if (JsonObject->TryGetObjectField(TEXT("DynamicData"), DynamicPtr))
            {
                DynamicData = *DynamicPtr;
            }
        }
    }

    // Called after standard export. Use it to add non-UPROPERTY fields to the JSON object.
    TSharedPtr<FJsonObject> PostJsonExport(TSharedPtr<FJsonObject> JsonObject) const
    {
        if (JsonObject.IsValid() && DynamicData.IsValid())
        {
            JsonObject->SetObjectField(TEXT("DynamicData"), DynamicData);
        }
        return JsonObject;
    }
};
```

## Usage

Replace `FJsonObjectConverter` with `ChrisUE::FEnhancedJsonConverter` in your code.

```cpp
// Instead of this:
// FJsonObjectConverter::JsonStringToUStruct(JsonString, MyStruct);

// Use this:
ChrisUE::FEnhancedJsonConverter::JsonStringToUStruct(JsonString, MyStruct);
```

### Example JSON

With the `FMyData` struct above, the converter can now handle this JSON:

```json
{
    "Name": "TestItem",
    "DynamicData": {
        "CustomField": "CustomValue",
        "AnotherField": 42,
        "NestedObject": {
            "InnerData": "InnerValue"
        }
    }
}
```
The `Name` field is handled by the standard converter, and `DynamicData` is correctly processed by your `PostJsonImport` method.

## Partial Updates

A common requirement is to update an existing object with new data from a server without overwriting fields that aren't included in the JSON response. The `Partial...` functions are designed for this exact scenario.

They will only update the `UPROPERTY` fields that are present in the JSON, leaving all other fields in your struct with their existing values.

### Partial Update Example

Imagine you have a `PlayerData` struct and you only want to update the player's health.

```cpp
// Your existing player data struct
USTRUCT()
struct FPlayerData
{
    GENERATED_BODY()

    UPROPERTY() FString Name = TEXT("PlayerOne");
    UPROPERTY() int32 Level = 10;
    UPROPERTY() int32 Health = 100;
};

FPlayerData PlayerData; // Name="PlayerOne", Level=10, Health=100

// You receive a JSON response that only contains a health update
const FString JsonString = TEXT("{"Health": 85}");

// Use PartialJsonStringToUStruct to apply the update
ChrisUE::FEnhancedJsonConverter::PartialJsonStringToUStruct(JsonString, PlayerData);

// The result:
// PlayerData.Name is still "PlayerOne"
// PlayerData.Level is still 10
// PlayerData.Health is now 85
```
The `PostJsonImport` method is also called after a partial update, allowing you to handle dynamic data updates as well.

## Core API

All functions are static and available under the `ChrisUE::FEnhancedJsonConverter` namespace.

### Full Conversion

Converts the entire object. `UPROPERTY` fields not present in the JSON will be reset to their default values.

- `bool JsonStringToUStruct(const FString& JsonString, TStruct& OutStruct);`
- `bool UStructToJsonString(const TStruct& InStruct, FString& OutJsonString);`
- `bool JsonObjectToUStruct(const TSharedPtr<FJsonObject>& JsonObject, TStruct& OutStruct);`
- `TSharedPtr<FJsonObject> UStructToJsonObject(const TStruct& InStruct);`

### Partial Update

Only updates `UPROPERTY` fields that are present in the JSON. Fields not in the JSON are left unchanged.

- `bool PartialJsonStringToUStruct(const FString& JsonString, TStruct& OutStruct);`
- `bool PartialJsonObjectToUStruct(const TSharedPtr<FJsonObject>& JsonObject, TStruct& OutStruct);`


## How It Works

1.  **Standard Conversion**: First, it uses Unreal's built-in `FJsonObjectConverter` to process all `UPROPERTY` fields.
2.  **Post-Processing**: It then checks for and calls your `PostJsonImport`/`PostJsonExport` methods if they exist.
3.  **Compile-Time Detection**: The check for the post-processing methods is done at compile-time using `if constexpr`, meaning there is zero runtime overhead if your struct doesn't implement them.

## License

MIT License
