# Spring

Essential guides for building Spring Boot applications with best practices.

## Web Development

### [HTTP Controller Best Practices](http_controller_best_practices.md)
Master building RESTful controllers with proper HTTP semantics. Covers:
- PATCH request implementation patterns
- Handling partial updates with nullable DTOs
- Explicit null value handling
- JSON Merge Patch (RFC 7396)
- Validation on partial updates
- Proper HTTP status codes
- Idempotency considerations
- DTO vs entity patterns

## Quick Reference

### PATCH with Nullable DTO
```kotlin
data class UserPatchDto(
    @field:Email
    val email: String? = null,
    @field:Size(min = 2, max = 50)
    val name: String? = null
)

@PatchMapping("/{id}")
fun updateUser(
    @PathVariable id: Long,
    @RequestBody @Valid patch: UserPatchDto
): User {
    val user = userRepository.findById(id).orElseThrow()
    patch.email?.let { user.email = it }
    patch.name?.let { user.name = it }
    return userRepository.save(user)
}
```

### HTTP Method Usage
- **GET**: Read, no side effects, cacheable
- **POST**: Create new resource, non-idempotent
- **PUT**: Full replacement, idempotent
- **PATCH**: Partial update, idempotent
- **DELETE**: Remove resource, idempotent

### Response Status Codes
```kotlin
// 200 OK - successful GET, PATCH, PUT
@GetMapping("/{id}")
fun getUser(@PathVariable id: Long): ResponseEntity<User> =
    ResponseEntity.ok(user)

// 201 Created - successful POST
@PostMapping
fun createUser(@RequestBody user: User): ResponseEntity<User> =
    ResponseEntity.created(URI.create("/users/${user.id}")).body(user)

// 204 No Content - successful DELETE
@DeleteMapping("/{id}")
@ResponseStatus(HttpStatus.NO_CONTENT)
fun deleteUser(@PathVariable id: Long) =
    userRepository.deleteById(id)
```
