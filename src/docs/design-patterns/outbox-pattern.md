---
title: "Outbox Pattern"
date: "2024-08-08"
description: The Outbox Pattern ensures reliable message delivery in distributed systems by storing outgoing messages in the same transaction as business data, guaranteeing eventual consistency and delivery.
featuredImage: "./images/outbox-pattern.png"
---

**Last Updated: August 2024**

The Outbox Pattern is a data management design pattern that ensures reliable message delivery in distributed systems. It addresses the challenge of maintaining consistency between updating business data and sending messages (such as events or notifications) by storing both operations in the same transactional boundary. This pattern is particularly valuable when you need to guarantee that messages are eventually delivered, even in the face of system failures.

## The Problem

In distributed systems, you often need to update your local database and send a message to other services or external systems (like sending an email, publishing an event, or calling an API). This creates a dual-write problem:

1. Update the business data in your database
2. Send a message/notification to external systems

If either operation fails, you can end up with inconsistent state. For example, you might successfully update your database but fail to send an email notification, leaving the user unaware of the change.

## The Solution

The Outbox Pattern solves this by:

1. **Single Transaction**: Store both the business data changes AND the outgoing message in the same database transaction
2. **Outbox Table**: Create a dedicated table to store pending outgoing messages
3. **Background Processor**: Use a separate process to read from the outbox table and deliver messages
4. **Idempotency**: Ensure message delivery is idempotent to handle potential duplicates

This approach guarantees that if the business operation succeeds, the message will eventually be delivered.

## Benefits

- **Reliability**: Messages are guaranteed to be delivered eventually
- **Consistency**: Business data and message delivery are kept in sync
- **Resilience**: System can recover from failures and continue processing
- **Decoupling**: Message delivery logic is separated from business logic
- **Observability**: All outgoing messages are tracked in the database

## Implementation with Entity Framework and C#

Here's a complete example implementing the Outbox Pattern for reliable email delivery:

### Outbox Entity

```csharp
public class OutboxMessage
{
    public Guid Id { get; set; }
    public string Type { get; set; } = string.Empty;
    public string Content { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public bool IsProcessed { get; set; }
    public int RetryCount { get; set; }
    public string? Error { get; set; }
}

public class EmailOutboxMessage
{
    public string To { get; set; } = string.Empty;
    public string Subject { get; set; } = string.Empty;
    public string Body { get; set; } = string.Empty;
    public string? AttachmentPath { get; set; }
}
```

### DbContext Configuration

```csharp
public class ApplicationDbContext : DbContext
{
    public DbSet<Customer> Customers { get; set; }
    public DbSet<Order> Orders { get; set; }
    public DbSet<OutboxMessage> OutboxMessages { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<OutboxMessage>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Type).HasMaxLength(255);
            entity.Property(e => e.Content).HasColumnType("ntext");
            entity.HasIndex(e => new { e.IsProcessed, e.CreatedAt });
        });
    }
}
```

### Service Implementation

```csharp
public interface IOrderService
{
    Task<Result> CreateOrderAsync(CreateOrderRequest request);
}

public class OrderService : IOrderService
{
    private readonly ApplicationDbContext _context;

    public OrderService(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<Result> CreateOrderAsync(CreateOrderRequest request)
    {
        using var transaction = await _context.Database.BeginTransactionAsync();
        
        try
        {
            // 1. Create the business entity
            var order = new Order
            {
                CustomerId = request.CustomerId,
                Total = request.Total,
                CreatedAt = DateTime.UtcNow
            };
            
            _context.Orders.Add(order);
            
            // 2. Create outbox message for email notification
            var emailMessage = new EmailOutboxMessage
            {
                To = request.CustomerEmail,
                Subject = "Order Confirmation",
                Body = $"Your order #{order.Id} has been created successfully."
            };
            
            var outboxMessage = new OutboxMessage
            {
                Id = Guid.NewGuid(),
                Type = "email",
                Content = JsonSerializer.Serialize(emailMessage),
                CreatedAt = DateTime.UtcNow,
                IsProcessed = false
            };
            
            _context.OutboxMessages.Add(outboxMessage);
            
            // 3. Save both in the same transaction
            await _context.SaveChangesAsync();
            await transaction.CommitAsync();
            
            return Result.Success();
        }
        catch (Exception ex)
        {
            await transaction.RollbackAsync();
            return Result.Failure($"Failed to create order: {ex.Message}");
        }
    }
}
```

### Background Processor

```csharp
public class OutboxProcessor : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<OutboxProcessor> _logger;

    public OutboxProcessor(IServiceProvider serviceProvider, ILogger<OutboxProcessor> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await ProcessOutboxMessages();
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing outbox messages");
            }
            
            await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
        }
    }

    private async Task ProcessOutboxMessages()
    {
        using var scope = _serviceProvider.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        var emailService = scope.ServiceProvider.GetRequiredService<IEmailService>();

        var unprocessedMessages = await context.OutboxMessages
            .Where(m => !m.IsProcessed && m.RetryCount < 3)
            .OrderBy(m => m.CreatedAt)
            .Take(10)
            .ToListAsync();

        foreach (var message in unprocessedMessages)
        {
            try
            {
                if (message.Type == "email")
                {
                    var emailMessage = JsonSerializer.Deserialize<EmailOutboxMessage>(message.Content);
                    await emailService.SendEmailAsync(emailMessage.To, emailMessage.Subject, emailMessage.Body);
                }

                // Mark as processed
                message.IsProcessed = true;
                message.ProcessedAt = DateTime.UtcNow;
                message.Error = null;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process outbox message {MessageId}", message.Id);
                
                message.RetryCount++;
                message.Error = ex.Message;
                
                // Stop retrying after 3 attempts
                if (message.RetryCount >= 3)
                {
                    _logger.LogError("Giving up on outbox message {MessageId} after 3 retries", message.Id);
                }
            }
        }

        await context.SaveChangesAsync();
    }
}
```

### Dependency Injection Setup

```csharp
// In Program.cs or Startup.cs
services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString));

services.AddScoped<IOrderService, OrderService>();
services.AddScoped<IEmailService, EmailService>();
services.AddHostedService<OutboxProcessor>();
```

## When to Use

The Outbox Pattern is particularly useful when:

- You need to guarantee message delivery in distributed systems
- You're working with critical business processes (orders, payments, notifications)
- You need to maintain consistency between database updates and external API calls
- You want to improve system resilience and reliability
- You need audit trails of all outgoing messages

## Considerations

- **Storage Overhead**: Additional storage required for outbox messages
- **Eventual Consistency**: Messages are delivered eventually, not immediately
- **Processing Delay**: There may be a delay between the business operation and message delivery
- **Cleanup**: Consider implementing cleanup processes for old processed messages
- **Monitoring**: Implement monitoring for failed messages and retry attempts

## References

- [Building a Resilient Email Method with .NET, Retry, and Outbox Pattern](https://ardalis.com/building-resilient-email-method-dotnet-retry-outbox-pattern)
- [YouTube: Outbox Pattern Explained](https://www.youtube.com/watch?v=qD3ZMH5x3uc)
- [Microservices.io: Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html)
- [Event Sourcing and CQRS](https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)