# JPA Query Optimization

Essential patterns for optimizing JPA queries with collections.

## Core Principles

### 1. Default Lazy Loading with @BatchSize

**Always use lazy loading for collections:**

```kotlin
@Entity
class Product(
    @OneToMany(
        mappedBy = "product",
        fetch = FetchType.LAZY
    )
    @BatchSize(size = 50)
    var reviews: MutableSet<Review> = mutableSetOf()
)
```

**Why:**
- Prevents loading unnecessary data
- @BatchSize mitigates N+1 problems
- Provides flexibility for different use cases

**Batch size selection:**
- Small (10-20): Lower memory, more queries
- Medium (40-100): Balanced (recommended)
- Large (500+): Fewer queries, higher memory

### 2. Avoid Eager Loading

```kotlin
// ❌ BAD - Always loads everything
@OneToMany(fetch = FetchType.EAGER)
var reviews: Set<Review> = setOf()

// ✅ GOOD - Load on demand
@OneToMany(fetch = FetchType.LAZY)
@BatchSize(size = 50)
var reviews: MutableSet<Review> = mutableSetOf()
```

**Problems with EAGER:**
- Cannot be overridden at runtime
- Multiple eager collections create Cartesian products
- Loads data even when not needed

## Query Strategies

### When to Modify Entities

Use JPQL with JOIN FETCH:

```kotlin
@Query("""
    select distinct p from Product p
    left join fetch p.reviews
    where p.id = :id
""")
fun findByIdWithReviews(id: Long): Product?
```

**Key points:**
- Use `DISTINCT` to avoid duplicates
- Only fetch collections you actually need
- Wrap in `@Transactional`

### Read-Only Operations

Use projections for better performance:

```kotlin
@Query(
    nativeQuery = true,
    value = """
        select
            p.id as productId,
            p.name as productName,
            p.price as price
        from products p
        where p.category_id = :categoryId
    """
)
fun findProductSummaries(categoryId: Long): List<ProductSummary>

interface ProductSummary {
    val productId: Long
    val productName: String
    val price: BigDecimal
}
```

**Benefits:**
- Faster - only loads needed columns
- No entity management overhead
- Great for DTOs and API responses

## N+1 Query Problem

### The Problem

```kotlin
// ❌ BAD - Triggers 1 + N queries
val products = productRepository.findAll()
products.forEach { product ->
    println(product.reviews.size) // Triggers query per product!
}
```

### Solution 1: @BatchSize (Already Applied)

```kotlin
@BatchSize(size = 50) // Reduces N queries to N/50 queries
var reviews: MutableSet<Review> = mutableSetOf()
```

### Solution 2: JOIN FETCH

```kotlin
@Query("""
    select distinct p from Product p
    left join fetch p.reviews
    where p.id in :ids
""")
fun findByIdsWithReviews(ids: List<Long>): List<Product>
```

### Solution 3: Separate Bulk Query

```kotlin
// Query 1: Get products
val products = productRepository.findByCategory(categoryId)

// Query 2: Get all reviews in one query
val allReviews = reviewRepository.findByProductIds(products.map { it.id })
val reviewsByProduct = allReviews.groupBy { it.productId }

// Combine in memory
```

## Cartesian Product Problem

### The Problem

```kotlin
// ❌ BAD - Creates M × N rows!
@Query("""
    select p from Product p
    left join fetch p.reviews
    left join fetch p.images
    where p.id = :id
""")
```

If a product has 5 reviews and 3 images, you get 15 rows!

### Solution: Separate Queries

```kotlin
// Query 1: Product with reviews
@Query("""
    select distinct p from Product p
    left join fetch p.reviews
    where p.id = :id
""")
fun findWithReviews(id: Long): Product?

// Query 2: Product with images
@Query("""
    select distinct p from Product p
    left join fetch p.images
    where p.id = :id
""")
fun findWithImages(id: Long): Product?
```

## Projections with Collections

### The Limitation

Native SQL projections **cannot contain collections directly**:

```kotlin
// ❌ This does NOT work
interface ProductProjection {
    val name: String
    val reviews: List<ReviewProjection> // Cannot do this!
}
```

### Solution: Separate Queries

```kotlin
// Query 1: Product projection
val product = productRepository.findProductSummary(id)

// Query 2: Reviews projection
val reviews = reviewRepository.findByProductId(id)

// Combine in service
return ProductWithReviewsDto(
    name = product.name,
    reviews = reviews
)
```

### Example Implementation

```kotlin
@Service
class ProductService(
    private val productRepository: ProductRepository,
    private val reviewRepository: ReviewRepository
) {
    fun getProductDetails(id: Long): ProductDetailsDto {
        // Query 1: Product data
        val product = productRepository.findProductSummary(id)
            ?: throw NotFoundException()

        // Query 2: Reviews
        val reviews = reviewRepository.findReviewsByProductId(id)

        // Combine
        return ProductDetailsDto(
            id = product.productId,
            name = product.productName,
            price = product.price,
            reviews = reviews.map { ReviewDto(it.rating, it.comment) }
        )
    }
}
```

## Bulk Operations

For multiple entities, avoid N+1 with IN clause:

```kotlin
@Query(
    nativeQuery = true,
    value = """
        select
            r.product_id as productId,
            r.rating as rating,
            r.comment as comment
        from reviews r
        where r.product_id in :productIds
    """
)
fun findByProductIds(productIds: List<Long>): List<ReviewProjection>

// Usage
val allReviews = reviewRepository.findByProductIds(productIds)
val reviewsByProduct = allReviews.groupBy { it.productId }
```

## Transaction Management

Always use `@Transactional` when accessing lazy collections:

```kotlin
// ❌ BAD - LazyInitializationException
fun getReviews(productId: Long): List<String> {
    val product = productRepository.findById(productId).get()
    return product.reviews.map { it.comment } // Exception!
}

// ✅ GOOD
@Transactional(readOnly = true)
fun getReviews(productId: Long): List<String> {
    val product = productRepository.findById(productId).get()
    return product.reviews.map { it.comment }
}
```

## Decision Tree

```
Need to modify the entity?
├─ YES → Use JPQL
│   └─ Need collections?
│       ├─ YES → JOIN FETCH
│       └─ NO → Simple query
│
└─ NO (read-only)
    └─ Use Projections
        └─ Need collections?
            ├─ YES → Separate queries
            └─ NO → Single projection
```

## Performance Checklist

- [ ] All `@OneToMany` use `FetchType.LAZY`
- [ ] All collections have `@BatchSize` annotation
- [ ] Use JOIN FETCH only when collections are needed
- [ ] Use projections for read-only operations
- [ ] Bulk queries with IN clause for multiple entities
- [ ] `@Transactional` on methods accessing lazy collections
- [ ] `DISTINCT` with JOIN FETCH
- [ ] Database indexes on foreign keys

## Common Mistakes

### 1. Eager Loading Collections
```kotlin
// ❌ Bad
@OneToMany(fetch = FetchType.EAGER)
```

### 2. No @BatchSize
```kotlin
// ❌ Bad - Missing @BatchSize
@OneToMany(fetch = FetchType.LAZY)
var reviews: Set<Review> = setOf()
```

### 3. Loading Full Entities for Read Operations
```kotlin
// ❌ Bad - Loads everything
val products = productRepository.findAll()
return products.map { it.name }

// ✅ Good - Only loads names
@Query("select p.name from Product p")
fun findAllNames(): List<String>
```

### 4. Queries in Loops
```kotlin
// ❌ Bad - N queries
productIds.forEach { id ->
    reviewRepository.findByProductId(id)
}

// ✅ Good - 1 query
reviewRepository.findByProductIds(productIds)
```

### 5. Missing DISTINCT with JOIN FETCH
```kotlin
// ❌ Bad - Returns duplicates
select p from Product p
left join fetch p.reviews

// ✅ Good
select distinct p from Product p
left join fetch p.reviews
```

## Debugging Performance

### Enable SQL Logging

```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        use_sql_comments: true
```

### Look for These Patterns

**N+1 Queries:**
```sql
SELECT * FROM products WHERE ...;
SELECT * FROM reviews WHERE product_id = 1;
SELECT * FROM reviews WHERE product_id = 2;
-- Many sequential queries
```

**Fix:** Use @BatchSize, JOIN FETCH, or bulk queries

**Cartesian Products:**
```sql
-- Returns M × N rows instead of N rows
SELECT p.*, r.*, i.*
FROM products p
LEFT JOIN reviews r ON ...
LEFT JOIN images i ON ...
```

**Fix:** Use separate queries

## Additional Resources

- [Hibernate Documentation](https://hibernate.org/orm/documentation/)
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/reference/)
- [Vlad Mihalcea's Blog](https://vladmihalcea.com/tutorials/hibernate/) - Excellent resource for JPA performance
