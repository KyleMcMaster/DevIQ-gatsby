---
title: "Outbox Design Pattern"
date: "2025-08-04"
description: The Outbox design pattern is a messaging pattern that can be used to ensure that messages are delivered at least once, even if the system crashes.
featuredImage: "./images/outbox-pattern.png"
---

The Outbox design pattern is a messaging pattern that is commonly used in distributed systems to ensure consistency between a database and a message broker. This pattern is useful when you need to send messages to other systems or services, and you want to ensure consistency between the message and the state of the system. One approach to the Outbox pattern works by writing the message to an "outbox" table in the database as part of the same transaction that updates the database state. A separate process then reads from the outbox table and sends the messages to the message broker. This can be useful for different integrations with external dependencies alongside a business application, like sending emails, events, and notifications.

## The Problem

In a distributed system, ensuring that messages are delivered reliably and consistently can be challenging. If a system crashes or a network failure occurs, messages may be lost or not processed correctly. This can lead to inconsistencies between the database state and the messages sent to other systems or services. The Outbox pattern helps to mitigate these issues by ensuring that messages are not lost and are sent exactly once. By writing messages to an outbox table as part of the same transaction that updates the database, the system can guarantee that messages are only sent if the database update is successful.


<TODO> mention zombie records and ghost messages

## Implementation Example

Here's a simple example using C# and Entity Framework:

```csharp
public class Order
{
    public int Id { get; set; }
    public string CustomerId { get; set; }
    public decimal Total { get; set; }
    public DateTime CreatedAt { get; set; }
}

public class OutboxMessage
{
    public int Id { get; set; }
    public string Type { get; set; }
    public string Data { get; set; }
    public DateTime CreatedAt { get; set; }
    public bool Processed { get; set; }
    public DateTime? ProcessedAt { get; set; }
}

public class OrderService
{
    private readonly ApplicationDbContext _context;
    
    public OrderService(ApplicationDbContext context)
    {
        _context = context;
    }
    
    public async Task CreateOrderAsync(CreateOrderRequest request)
    {
        using var transaction = await _context.Database.BeginTransactionAsync();
        
        try
        {
            // 1. Create the order
            var order = new Order
            {
                CustomerId = request.CustomerId,
                Total = request.Total,
                CreatedAt = DateTime.UtcNow
            };
            
            _context.Orders.Add(order);
            await _context.SaveChangesAsync();
            
            // 2. Add message to outbox (same transaction)
            var outboxMessage = new OutboxMessage
            {
                Type = "OrderCreated",
                Data = JsonSerializer.Serialize(new 
                {
                    OrderId = order.Id,
                    CustomerId = order.CustomerId,
                    Total = order.Total
                }),
                CreatedAt = DateTime.UtcNow,
                Processed = false
            };
            
            _context.OutboxMessages.Add(outboxMessage);
            await _context.SaveChangesAsync();
            
            // 3. Commit both operations atomically
            await transaction.CommitAsync();
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

### Background Message Processor

The background process that publishes messages from the outbox:

```csharp
public class OutboxMessageProcessor : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly IMessagePublisher _messagePublisher;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await ProcessPendingMessagesAsync();
                await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
            }
            catch (Exception ex)
            {
                // Log error and continue
                Console.WriteLine($"Error processing outbox messages: {ex.Message}");
            }
        }
    }
    
    private async Task ProcessPendingMessagesAsync()
    {
        using var scope = _serviceProvider.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        
        var pendingMessages = await context.OutboxMessages
            .Where(m => !m.Processed)
            .OrderBy(m => m.CreatedAt)
            .Take(100)
            .ToListAsync();
            
        foreach (var message in pendingMessages)
        {
            try
            {
                await _messagePublisher.PublishAsync(message.Type, message.Data);
                
                message.Processed = true;
                message.ProcessedAt = DateTime.UtcNow;
                
                await context.SaveChangesAsync();
            }
            catch (Exception ex)
            {
                // Log error but don't mark as processed
                // Will be retried in next iteration
                Console.WriteLine($"Failed to publish message {message.Id}: {ex.Message}");
            }
        }
    }
}
```


## References 

- [Building a Resilient Email Sending Method in .NET with SmtpClient, Retry Support, and the Outbox Pattern](https://ardalis.com/building-resilient-email-method-dotnet-retry-outbox-pattern/)
- [Send Email in dotnet with Mimekit, Retry, and Outbox Pattern
](https://www.youtube.com/watch?v=qD3ZMH5x3uc)
- [NServiceBus Outbox Documentation](https://docs.particular.net/nservicebus/outbox/)