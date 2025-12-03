# HTTP Controller Best Practices

## PATCH Requests

PATCH is for partial updates. Unlike PUT (full replacement), PATCH only modifies specified fields.

### Basic Implementation

```kotlin
@PatchMapping("/{id}")
fun updateUser(
    @PathVariable id: Long,
    @RequestBody updates: Map<String, Any>
): User {
    val user = userRepository.findById(id).orElseThrow()
    // Apply updates manually - not recommended for production
    return userRepository.save(user)
}
```

### Better: Use a DTO with Nullable Fields

```kotlin
data class UserPatchDto(
    val email: String? = null,
    val name: String? = null,
    val age: Int? = null
)

@PatchMapping("/{id}")
fun updateUser(
    @PathVariable id: Long,
    @RequestBody @Valid patch: UserPatchDto
): User {
    val user = userRepository.findById(id).orElseThrow()

    patch.email?.let { user.email = it }
    patch.name?.let { user.name = it }
    patch.age?.let { user.age = it }

    return userRepository.save(user)
}
```

**Key principle**: `null` means "don't update", absence means "don't update", explicit value means "update to this value".

### Handling Explicit Null (Setting Field to Null)

The problem: With nullable DTOs, you cannot distinguish:
- Field not sent: `{}` → don't update email
- Field sent as null: `{"email": null}` → set email to null

**Solution: Use a custom deserializer with an Optional wrapper**

```kotlin
// Wrapper class to track if field was present
data class Optional<T>(val value: T?) {
    val isPresent: Boolean = true

    companion object {
        fun <T> absent() = Optional<T>(null).apply {
            // Use reflection or custom marker to track absence
        }
    }
}

// Custom deserializer
class OptionalDeserializer : JsonDeserializer<Optional<*>>() {
    override fun deserialize(p: JsonParser, ctxt: DeserializationContext): Optional<*> {
        return if (p.currentToken == JsonToken.VALUE_NULL) {
            Optional(null)  // Field present, value is null
        } else {
            Optional(p.readValueAs(Any::class.java))
        }
    }
}

data class UserPatchDto(
    @JsonDeserialize(using = OptionalDeserializer::class)
    val email: Optional<String>? = null,

    @JsonDeserialize(using = OptionalDeserializer::class)
    val name: Optional<String>? = null
)

@PatchMapping("/{id}")
fun updateUser(
    @PathVariable id: Long,
    @RequestBody patch: UserPatchDto
): User {
    val user = userRepository.findById(id).orElseThrow()

    // null = field not sent, don't update
    // Optional(null) = field sent as null, set to null
    // Optional(value) = field sent with value, update
    patch.email?.let { user.email = it.value }
    patch.name?.let { user.name = it.value }

    return userRepository.save(user)
}
```

**Request examples:**
```json
// Don't update email
{"name": "John"}

// Set email to null
{"email": null, "name": "John"}

// Update email
{"email": "new@example.com", "name": "John"}
```

**Simpler alternative using Kotlin's reflection:**

```kotlin
import kotlin.reflect.full.memberProperties

data class UserPatchDto(
    val email: String? = null,
    val name: String? = null,
    val age: Int? = null
)

@PatchMapping("/{id}")
fun updateUser(
    @PathVariable id: Long,
    @RequestBody patch: UserPatchDto,
    request: HttpServletRequest
): User {
    val user = userRepository.findById(id).orElseThrow()

    // Read raw JSON to check which fields were actually sent
    val jsonBody = request.reader.readText()
    val sentFields = ObjectMapper().readTree(jsonBody).fieldNames().asSequence().toSet()

    if ("email" in sentFields) user.email = patch.email
    if ("name" in sentFields) user.name = patch.name
    if ("age" in sentFields) user.age = patch.age

    return userRepository.save(user)
}
```

### Using Jackson for Cleaner Null Handling

```kotlin
@PatchMapping("/{id}")
fun updateUser(
    @PathVariable id: Long,
    @RequestBody updates: JsonNode
): User {
    val user = userRepository.findById(id).orElseThrow()

    updates.fields().forEach { (key, value) ->
        when (key) {
            "email" -> user.email = if (value.isNull) null else value.asText()
            "name" -> user.name = if (value.isNull) null else value.asText()
            "age" -> user.age = if (value.isNull) null else value.asInt()
        }
    }

    return userRepository.save(user)
}
```

### JSON Merge Patch (RFC 7396)

Spring supports JSON Merge Patch with the correct content type:

```kotlin
@PatchMapping("/{id}", consumes = ["application/merge-patch+json"])
fun mergePatchUser(
    @PathVariable id: Long,
    @RequestBody patch: UserPatchDto
): User {
    val user = userRepository.findById(id).orElseThrow()

    // With merge-patch, explicit null means "set to null"
    patch.email?.let { user.email = it }
    patch.name?.let { user.name = it }

    return userRepository.save(user)
}
```

### Validation on PATCH

```kotlin
data class UserPatchDto(
    @field:Email
    val email: String? = null,

    @field:Size(min = 2, max = 50)
    val name: String? = null,

    @field:Min(0)
    @field:Max(150)
    val age: Int? = null
)
```

Validation only runs on fields that are present in the request.

## General Controller Best Practices

### Response Codes

```kotlin
@PatchMapping("/{id}")
fun updateUser(@PathVariable id: Long, @RequestBody patch: UserPatchDto): ResponseEntity<User> {
    val user = userRepository.findById(id).orElseThrow()
    // apply updates
    return ResponseEntity.ok(userRepository.save(user))
}

@PostMapping
fun createUser(@RequestBody @Valid user: User): ResponseEntity<User> {
    val created = userRepository.save(user)
    return ResponseEntity
        .created(URI.create("/users/${created.id}"))
        .body(created)
}

@DeleteMapping("/{id}")
@ResponseStatus(HttpStatus.NO_CONTENT)
fun deleteUser(@PathVariable id: Long) {
    userRepository.deleteById(id)
}
```

### Idempotency

- **GET, PUT, DELETE**: Naturally idempotent
- **PATCH**: Should be idempotent (applying same patch multiple times = same result)
- **POST**: Not idempotent (creates new resource each time)

### Error Handling

```kotlin
@ExceptionHandler(EntityNotFoundException::class)
fun handleNotFound(e: EntityNotFoundException) =
    ResponseEntity.status(HttpStatus.NOT_FOUND).body(ErrorResponse(e.message))

@ExceptionHandler(MethodArgumentNotValidException::class)
fun handleValidation(e: MethodArgumentNotValidException) =
    ResponseEntity.badRequest().body(
        ValidationErrorResponse(
            e.bindingResult.fieldErrors.associate { it.field to it.defaultMessage }
        )
    )
```

### Don't Return Entities Directly

Use DTOs to avoid:
- Lazy loading issues (Jackson serialization triggering queries)
- Exposing internal structure
- Circular references

```kotlin
@GetMapping("/{id}")
fun getUser(@PathVariable id: Long): UserDto {
    return userRepository.findById(id)
        .orElseThrow()
        .toDto()
}

fun User.toDto() = UserDto(
    id = id,
    email = email,
    name = name
)
```

### Use Proper HTTP Methods

- **GET**: Read, no side effects, cacheable
- **POST**: Create, non-idempotent
- **PUT**: Full replacement, idempotent
- **PATCH**: Partial update, idempotent
- **DELETE**: Remove, idempotent

### Content Negotiation

```kotlin
@GetMapping("/{id}", produces = [MediaType.APPLICATION_JSON_VALUE])
fun getUserJson(@PathVariable id: Long): UserDto = // ...

@GetMapping("/{id}", produces = [MediaType.APPLICATION_XML_VALUE])
fun getUserXml(@PathVariable id: Long): UserDto = // ...
```

Or let Spring handle it automatically based on `Accept` header.
