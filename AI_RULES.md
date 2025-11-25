# AI Rules for Development

This document contains comprehensive guidelines and rules for AI-assisted development in this project.

## 1. Exception Handling

### Guidelines
- **Use `BusinessException` for business rule violations** - Always throw `BusinessException` when business logic constraints are violated.
- **Keep exception usage simple and clear** - Don't overcomplicate exception hierarchies; use straightforward exception patterns.
- **Handle errors gracefully** - Avoid leaking stack traces to the client; provide user-friendly error messages instead.
- **Use `log.error` when needed** - Particularly inside `doOnError` blocks for reactive streams.
- **Use descriptive exception messages** - Exception messages should clearly explain what went wrong and ideally why.

### Example
```java
if (order.getAmount() <= 0) {
    throw new BusinessException("Order amount must be greater than zero");
}
```

---

## 2. Logging Standards

### Guidelines
- **Use `EXTLogger` for logging** - Follow the system pattern for consistent logging across the application.
- **Log errors using `log.error()`** - Use this method when logging exceptions and error conditions.
- **Avoid excessive logging** - Only log useful events; don't clutter logs with redundant information.

### Example
```java
try {
    processOrder(order);
} catch (Exception e) {
    log.error("Failed to process order: {}", order.getId(), e);
    throw new BusinessException("Order processing failed");
}
```

---

## 3. Service & Controller Architecture

### Guidelines
- **Keep all business logic in services, not controllers** - Controllers should only handle HTTP concerns.
- **Controllers should be thin** - They should focus on:
  - Request mapping
  - Input validation
  - Delegation to services
- **Keep service methods single-responsibility and focused** - Each method should do one thing well.
- **Avoid static methods unless they are stateless utilities** - Instance methods allow for better testability and dependency injection.

### Example
```java
// Controller - thin layer
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    @PostMapping
    public Mono<OrderDTO> createOrder(@Valid @RequestBody OrderDTO orderDTO) {
        return orderService.createOrder(orderDTO);
    }
}

// Service - contains business logic
@Service
public class OrderService {
    
    public Mono<OrderDTO> createOrder(OrderDTO orderDTO) {
        // Business logic here
    }
}
```

---

## 4. Coding Style & Clean Code

### Guidelines
- **Use meaningful method and variable names** - Names should clearly express intent.
- **Document any non-trivial side effects or assumptions** - Use comments to explain why, not what.
- **Validate parameters using JSR-303 annotations** - Apply validation annotations where applicable (`@NotNull`, `@Min`, etc.).
- **Use method-level Javadoc for complex logic** - Document complex methods to help future maintainers.
- **Ensure all dependencies are imported and available** - Verify all required imports are present.
- **Use `CollectionUtils.isEmpty()` to simplify list null/empty checks** - This handles both null and empty cases.

### Example
```java
/**
 * Processes the order and updates inventory.
 * This method has the side effect of sending a notification email.
 * 
 * @param orderId the ID of the order to process
 * @return the processed order
 */
public Order processOrder(@NotNull Long orderId) {
    Order order = orderRepository.findById(orderId);
    if (CollectionUtils.isEmpty(order.getItems())) {
        throw new BusinessException("Order must contain at least one item");
    }
    // Process order...
    return order;
}
```

---

## 5. Reactive Programming

### Guidelines
- **Use reactive types (`Mono`, `Flux`) only for I/O-bound operations** - Don't use reactive types for CPU-bound or simple operations.
- **Avoid blocking operations inside reactive pipelines** - Blocking defeats the purpose of reactive programming.
- **Use `doOnError` with `log.error` to log reactive errors** - Ensure errors in reactive streams are logged.
- **Do not use reactive types in DTOs** - DTOs should contain only plain data types.

### Example
```java
public Mono<Order> getOrder(Long orderId) {
    return orderRepository.findById(orderId)
        .doOnError(e -> log.error("Error fetching order: {}", orderId, e))
        .switchIfEmpty(Mono.error(new BusinessException("Order not found")));
}
```

---

## 6. DTO & Model Guidelines

### Guidelines
- **DTOs must contain only data, no business logic or I/O** - DTOs are for data transfer only.
- **Keep DTOs simple and well-structured** - Clear, focused data structures.
- **Use Lombok annotations** - `@Data`, `@AllArgsConstructor`, `@NoArgsConstructor` for cleaner code.
- **Validate fields using JSR-303** - Use annotations like `@NotNull`, `@Min`, `@Max`, `@Size`, etc.
- **Use default field values where applicable** - Provide sensible defaults.
- **Ensure all referenced classes are imported and available** - Verify dependencies.

### Example
```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class OrderDTO {
    
    private Long id;
    
    @NotNull(message = "Customer ID is required")
    private Long customerId;
    
    @NotNull(message = "Items list is required")
    @Size(min = 1, message = "Order must contain at least one item")
    private List<OrderItemDTO> items;
    
    @Min(value = 0, message = "Amount must be non-negative")
    private BigDecimal amount = BigDecimal.ZERO;
}
```

---

## 7. Testing

### Guidelines
- **Write unit tests for all public service methods** - Ensure comprehensive test coverage.
- **Use `StepVerifier` for testing reactive behavior** - Test `Mono` and `Flux` properly with StepVerifier.
- **Mock external dependencies properly** - Use Mockito to mock dependencies.
- **Use `@ExtendWith(MockitoExtension.class)` universally** - Standard test setup for JUnit 5 with Mockito.

### Example
```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    
    @Mock
    private OrderRepository orderRepository;
    
    @InjectMocks
    private OrderService orderService;
    
    @Test
    void testCreateOrder_Success() {
        // Arrange
        OrderDTO orderDTO = new OrderDTO();
        Order order = new Order();
        when(orderRepository.save(any())).thenReturn(Mono.just(order));
        
        // Act & Assert
        StepVerifier.create(orderService.createOrder(orderDTO))
            .expectNextMatches(result -> result != null)
            .verifyComplete();
    }
    
    @Test
    void testCreateOrder_WithError() {
        // Arrange
        OrderDTO orderDTO = new OrderDTO();
        when(orderRepository.save(any()))
            .thenReturn(Mono.error(new RuntimeException("DB Error")));
        
        // Act & Assert
        StepVerifier.create(orderService.createOrder(orderDTO))
            .expectError(BusinessException.class)
            .verify();
    }
}
```

---

## Summary

These rules ensure:
- **Consistency** - Code follows predictable patterns
- **Maintainability** - Code is easy to understand and modify
- **Quality** - Best practices are enforced
- **Testability** - Code is designed to be testable

Always refer to these guidelines when writing or reviewing code.
