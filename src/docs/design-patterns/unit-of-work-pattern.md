---
title: "Unit of Work Pattern"
date: "2024-12-15"
description: Learn about the Unit of Work design pattern, which maintains a list of objects affected by a business transaction and coordinates writing out changes and resolving concurrency problems.
featuredImage: "./images/unit-of-work-pattern.png"
---

The **Unit of Work Pattern** maintains a list of objects affected by a business transaction and coordinates writing out changes and resolving concurrency problems. It tracks changes made to a set of domain objects and ensures that all changes are committed together as a single transaction or all are rolled back if any operation fails.

## Purpose

The Unit of Work pattern serves several important purposes:

- **Transaction Management**: Ensures that all changes within a business operation are committed or rolled back together
- **Change Tracking**: Keeps track of which objects have been modified, added, or deleted during a business transaction
- **Performance Optimization**: Batches database operations to reduce the number of round trips to the database
- **Consistency**: Maintains data consistency by ensuring related changes are applied atomically

## How It Works

The Unit of Work pattern typically maintains three lists of objects:

- **New Objects**: Objects that need to be inserted into the database
- **Dirty Objects**: Objects that have been modified and need to be updated
- **Removed Objects**: Objects that need to be deleted from the database

When a business operation completes, the Unit of Work commits all changes in the appropriate order, handling any dependencies between objects.

## Relationship with Repository Pattern

The Unit of Work pattern is commonly used together with the [Repository Pattern](/design-patterns/repository-pattern). While the Repository pattern provides an abstraction for data access operations, the Unit of Work pattern manages the lifecycle of a business transaction:

- **Repository** handles individual CRUD operations for specific aggregate roots
- **Unit of Work** coordinates multiple repository operations within a single transaction

```csharp
public interface IUnitOfWork
{
    IRepository<Customer> Customers { get; }
    IRepository<Order> Orders { get; }
    IRepository<Product> Products { get; }
    
    Task<int> CommitAsync();
    void Rollback();
}
```

## Implementation Example

A typical Unit of Work implementation might look like this:

```csharp
public class UnitOfWork : IUnitOfWork, IDisposable
{
    private readonly DbContext _context;
    private IRepository<Customer> _customers;
    private IRepository<Order> _orders;
    private IRepository<Product> _products;

    public UnitOfWork(DbContext context)
    {
        _context = context;
    }

    public IRepository<Customer> Customers =>
        _customers ??= new Repository<Customer>(_context);

    public IRepository<Order> Orders =>
        _orders ??= new Repository<Order>(_context);

    public IRepository<Product> Products =>
        _products ??= new Repository<Product>(_context);

    public async Task<int> CommitAsync()
    {
        return await _context.SaveChangesAsync();
    }

    public void Rollback()
    {
        foreach (var entry in _context.ChangeTracker.Entries())
        {
            entry.State = EntityState.Unchanged;
        }
    }

    public void Dispose()
    {
        _context.Dispose();
    }
}
```

## Benefits

- **Atomicity**: All changes succeed or fail together
- **Performance**: Batches database operations for better performance
- **Consistency**: Maintains data integrity across related operations
- **Testability**: Makes it easier to test business logic that involves multiple data changes
- **[Separation of Concerns](/principles/separation-of-concerns)**: Separates transaction management from business logic

## Drawbacks

- **Complexity**: Adds another layer of abstraction that may not be needed for simple applications
- **Memory Usage**: May hold references to many objects during the transaction lifetime
- **ORM Integration**: Many modern ORMs like Entity Framework already provide Unit of Work functionality through their DbContext

## When to Use

Consider using the Unit of Work pattern when:

- You need to coordinate multiple related data operations
- You want to ensure transactional consistency across multiple repositories
- You need to optimize database performance by batching operations
- You're working with complex business transactions that span multiple aggregates

## Modern Considerations

Many modern Object-Relational Mapping (ORM) frameworks like Entity Framework Core already implement the Unit of Work pattern internally through their DbContext class. In such cases, you may not need to implement the pattern explicitly, but understanding it helps you work more effectively with these frameworks.

## See Also

- [Repository Pattern](/design-patterns/repository-pattern)
- [Specification Pattern](/design-patterns/specification-pattern)
- [Separation of Concerns](/principles/separation-of-concerns)

## References

- [Patterns of Enterprise Application Architecture](https://martinfowler.com/eaaCatalog/unitOfWork.html) by Martin Fowler
- [Enterprise Application Patterns - Unit of Work](https://www.martinfowler.com/eaaCatalog/unitOfWork.html)
- [Domain-Driven Design](http://bit.ly/PS-DDD) - includes discussion of aggregates and transaction boundaries