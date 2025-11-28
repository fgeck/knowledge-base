# SQL & Databases

Essential guides for working with SQL databases and JPA/Hibernate.

## JPA & Hibernate

### [JPA Query Optimization](jpa-optimization.md)
Learn essential patterns for optimizing JPA queries with collections. Covers:
- Lazy vs eager loading strategies
- The N+1 query problem and solutions
- Using @BatchSize effectively
- JOIN FETCH patterns
- Bulk operations
- Common mistakes to avoid

### [JPA Projections and Collections](jpa-projections.md)
Master using projections for read-only operations. Topics include:
- Interface projections with native SQL
- Working with collections in projections
- Avoiding N+1 queries with bulk operations
- Complete working examples
- When to use projections vs full entities

## Quick Reference

### Default Entity Setup
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

### Performance Checklist
- [ ] All `@OneToMany` use `FetchType.LAZY`
- [ ] All collections have `@BatchSize` annotation
- [ ] Use projections for read-only operations
- [ ] Use JOIN FETCH when modifying entities
- [ ] Bulk queries with IN clause
- [ ] Proper indexes on foreign keys

### Common Patterns

**Load entity with collections:**
```kotlin
@Query("""
    select distinct e from Entity e
    left join fetch e.children
    where e.id = :id
""")
fun findWithChildren(id: Long): Entity?
```

**Projection with separate queries:**
```kotlin
// Query 1: Parent
val parent = parentRepo.findSummary(id)

// Query 2: Children
val children = childRepo.findByParentId(id)

// Combine in service
return ParentDto(parent, children)
```

**Bulk query for multiple entities:**
```kotlin
@Query("""
    select c.parent_id as parentId, c.name as name
    from children c
    where c.parent_id in :parentIds
""")
fun findByParentIds(parentIds: List<Long>): List<ChildProjection>
```
