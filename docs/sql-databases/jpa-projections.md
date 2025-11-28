# JPA Projections and Collections

Guide to using projections effectively with Spring Data JPA.

## What Are Projections?

Projections allow you to load only specific fields instead of full entities, improving performance for read-only operations.

## Interface Projections

### Basic Example

```kotlin
@Query(
    nativeQuery = true,
    value = """
        select
            p.id as productId,
            p.name as productName,
            p.price as price,
            c.name as categoryName
        from products p
        inner join categories c on p.category_id = c.id
        where p.id = :id
    """
)
fun findProductDetails(id: Long): ProductDetails?

interface ProductDetails {
    val productId: Long
    val productName: String
    val price: BigDecimal
    val categoryName: String
}
```

### When to Use

✅ Read-only operations
✅ API responses and DTOs
✅ Performance-critical queries
✅ Only need specific fields
✅ Joining multiple tables

### Limitations

❌ Cannot contain collections directly
❌ Cannot modify and save back
❌ Only works with flat result sets

## Working with Collections

### The Problem

This does **NOT** work:

```kotlin
// ❌ WRONG - Cannot do this with native SQL projections
interface ProductDetails {
    val productName: String
    val reviews: List<ReviewProjection>
}
```

### Solution: Separate Queries

**Repository Layer:**

```kotlin
// Query 1: Product projection
@Query(
    nativeQuery = true,
    value = """
        select
            p.id as productId,
            p.name as productName,
            p.price as price
        from products p
        where p.id = :id
    """
)
fun findProductSummary(id: Long): ProductSummary?

interface ProductSummary {
    val productId: Long
    val productName: String
    val price: BigDecimal
}

// Query 2: Reviews projection
@Query(
    nativeQuery = true,
    value = """
        select
            r.id as reviewId,
            r.product_id as productId,
            r.rating as rating,
            r.comment as comment
        from reviews r
        where r.product_id = :productId
        order by r.created_date desc
    """
)
fun findReviewsByProductId(productId: Long): List<ReviewProjection>

interface ReviewProjection {
    val reviewId: Long
    val productId: Long
    val rating: Int
    val comment: String
}
```

**Service Layer:**

```kotlin
@Service
class ProductService(
    private val productRepository: ProductRepository,
    private val reviewRepository: ReviewRepository
) {
    fun getProductWithReviews(id: Long): ProductWithReviewsDto {
        // Query 1: Get product
        val product = productRepository.findProductSummary(id)
            ?: throw NotFoundException("Product not found: $id")

        // Query 2: Get reviews
        val reviews = reviewRepository.findReviewsByProductId(id)

        // Combine in DTO
        return ProductWithReviewsDto(
            productId = product.productId,
            productName = product.productName,
            price = product.price,
            reviews = reviews.map {
                ReviewDto(
                    id = it.reviewId,
                    rating = it.rating,
                    comment = it.comment
                )
            }
        )
    }
}

data class ProductWithReviewsDto(
    val productId: Long,
    val productName: String,
    val price: BigDecimal,
    val reviews: List<ReviewDto>
)

data class ReviewDto(
    val id: Long,
    val rating: Int,
    val comment: String
)
```

## Multiple Entities with Collections

### Avoiding N+1 Queries

```kotlin
@Service
class ProductService(
    private val productRepository: ProductRepository,
    private val reviewRepository: ReviewRepository
) {
    fun getMultipleProductsWithReviews(
        categoryId: Long
    ): List<ProductWithReviewsDto> {
        // Query 1: Get all products
        val products = productRepository.findByCategory(categoryId)

        if (products.isEmpty()) return emptyList()

        val productIds = products.map { it.productId }

        // Query 2: Get ALL reviews for ALL products in ONE query
        val allReviews = reviewRepository.findByProductIds(productIds)

        // Group reviews by product ID in memory
        val reviewsByProduct = allReviews.groupBy { it.productId }

        // Combine
        return products.map { product ->
            val reviews = reviewsByProduct[product.productId] ?: emptyList()

            ProductWithReviewsDto(
                productId = product.productId,
                productName = product.productName,
                price = product.price,
                reviews = reviews.map { ReviewDto(it.reviewId, it.rating, it.comment) }
            )
        }
    }
}

// Bulk query repository method
@Query(
    nativeQuery = true,
    value = """
        select
            r.id as reviewId,
            r.product_id as productId,
            r.rating as rating,
            r.comment as comment
        from reviews r
        where r.product_id in :productIds
        order by r.created_date desc
    """
)
fun findByProductIds(productIds: List<Long>): List<ReviewProjection>
```

**Key Points:**
- 2 queries total (not N+1)
- First query gets all products
- Second query gets all reviews using IN clause
- Group in memory using `groupBy`

## Class-Based Projections

Alternative to interface projections using JPQL:

```kotlin
data class ProductDto(
    val id: Long,
    val name: String,
    val price: BigDecimal,
    val categoryName: String
)

@Query("""
    select new com.example.ProductDto(
        p.id,
        p.name,
        p.price,
        p.category.name
    )
    from Product p
    where p.id = :id
""")
fun findProductDto(id: Long): ProductDto?
```

**When to Use:**
- Need immutable DTOs
- Type safety is important
- JPQL is sufficient (not native SQL)

**Limitations:**
- Only works with JPQL (not native SQL)
- Cannot contain collections

## Projection Comparison

| Feature | Interface | Class DTO | Full Entity |
|---------|-----------|-----------|-------------|
| **SQL Type** | Native or JPQL | JPQL only | JPQL |
| **Collections** | ❌ No | ❌ No | ✅ Yes |
| **Performance** | ⚡ Fast | ⚡ Fast | Slower |
| **Modifiable** | ❌ No | ❌ No | ✅ Yes |
| **Use Case** | Read-only | Read-only DTOs | Modifications |

## Complete Example

Here's a full working example with generic entities:

```kotlin
// Entities
@Entity
@Table(name = "products")
class Product(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    var name: String,
    var price: BigDecimal,

    @ManyToOne
    @JoinColumn(name = "category_id")
    var category: Category,

    @OneToMany(
        mappedBy = "product",
        fetch = FetchType.LAZY
    )
    @BatchSize(size = 50)
    var reviews: MutableSet<Review> = mutableSetOf()
)

@Entity
@Table(name = "reviews")
class Review(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @ManyToOne
    @JoinColumn(name = "product_id")
    var product: Product,

    var rating: Int,
    var comment: String,
    var createdDate: LocalDateTime = LocalDateTime.now()
)

// Repository
interface ProductRepository : JpaRepository<Product, Long> {

    @Query(
        nativeQuery = true,
        value = """
            select
                p.id as productId,
                p.name as productName,
                p.price as price,
                c.name as categoryName
            from products p
            inner join categories c on p.category_id = c.id
            where p.category_id = :categoryId
        """
    )
    fun findProductsByCategory(categoryId: Long): List<ProductSummary>

    interface ProductSummary {
        val productId: Long
        val productName: String
        val price: BigDecimal
        val categoryName: String
    }
}

interface ReviewRepository : JpaRepository<Review, Long> {

    @Query(
        nativeQuery = true,
        value = """
            select
                r.id as reviewId,
                r.product_id as productId,
                r.rating as rating,
                r.comment as comment,
                r.created_date as createdDate
            from reviews r
            where r.product_id in :productIds
            order by r.created_date desc
        """
    )
    fun findByProductIds(productIds: List<Long>): List<ReviewSummary>

    interface ReviewSummary {
        val reviewId: Long
        val productId: Long
        val rating: Int
        val comment: String
        val createdDate: LocalDateTime
    }
}

// Service
@Service
class ProductService(
    private val productRepository: ProductRepository,
    private val reviewRepository: ReviewRepository
) {

    fun getProductsByCategoryWithReviews(
        categoryId: Long
    ): List<ProductWithReviewsDto> {
        // Query 1: Get products
        val products = productRepository.findProductsByCategory(categoryId)

        if (products.isEmpty()) return emptyList()

        // Query 2: Get all reviews in one query
        val productIds = products.map { it.productId }
        val allReviews = reviewRepository.findByProductIds(productIds)
        val reviewsByProduct = allReviews.groupBy { it.productId }

        // Combine
        return products.map { product ->
            ProductWithReviewsDto(
                id = product.productId,
                name = product.productName,
                price = product.price,
                categoryName = product.categoryName,
                reviews = reviewsByProduct[product.productId]
                    ?.map { ReviewDto(it.reviewId, it.rating, it.comment) }
                    ?: emptyList()
            )
        }
    }
}

// DTOs
data class ProductWithReviewsDto(
    val id: Long,
    val name: String,
    val price: BigDecimal,
    val categoryName: String,
    val reviews: List<ReviewDto>
)

data class ReviewDto(
    val id: Long,
    val rating: Int,
    val comment: String
)
```

## Alternative: JPQL with JOIN FETCH

If you don't need native SQL, JPQL with JOIN FETCH is simpler:

```kotlin
@Query("""
    select distinct p from Product p
    join fetch p.category
    left join fetch p.reviews
    where p.id = :id
""")
fun findByIdWithAll(id: Long): Product?
```

**Pros:**
- Single query gets everything
- Returns full entities
- Can modify and save

**Cons:**
- Loads full entities (more data)
- Entity management overhead
- Not suitable for read-only DTOs

## Best Practices

1. **Use projections for read-only operations**
   - Faster queries
   - Less memory
   - Better for APIs

2. **Combine separate queries in service layer**
   - Repositories stay simple
   - Easy to test
   - Clear separation of concerns

3. **Use bulk queries with IN clause**
   - Avoids N+1 problems
   - Efficient for multiple entities
   - Group results in memory

4. **Keep DTOs simple**
   - Data classes work great
   - Immutable when possible
   - Match your API needs

5. **Index foreign keys**
   - Essential for good performance
   - Especially for IN clauses
   - Check query plans regularly

## Summary

**For collections with projections:**
- ✅ Use separate queries
- ✅ Combine in service layer
- ✅ Use bulk queries for multiple entities
- ✅ Group results in memory
- ❌ Don't try to include collections in projections
- ❌ Don't query in loops

**Performance comparison:**
- Interface projections: 2 queries, minimal data
- JPQL JOIN FETCH: 1 query, full entities
- No optimization: N+1 queries, disaster!
