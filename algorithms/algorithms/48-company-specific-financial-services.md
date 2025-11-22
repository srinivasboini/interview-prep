# Company-Specific: Financial Services - Interview Guide

## Overview
Banking/finance interviews emphasize correctness, edge cases, concurrency, and domain knowledge.

---

## Interview Focus
- **Correctness**: No bugs in financial calculations
- **Edge Cases**: Handle all scenarios (negative amounts, overflow)
- **Concurrency**: Thread-safe code
- **Performance**: Low latency for trading systems
- **Domain Knowledge**: Financial concepts

---

## Common Problem Types

### 1. Transaction Processing
- **LRU Cache**: Session management
- **Merge K Sorted Lists**: Merge transaction logs
- **Top K Frequent**: Most active accounts
- **Two Sum**: Find transaction pairs

### 2. Time Series Analysis
- **Sliding Window**: Rolling metrics (30-day average)
- **Stock Price Problems**: Max profit, cooldown
- **Median from Stream**: Real-time analytics

### 3. Risk & Compliance
- **Graph Algorithms**: Detect circular payments (money laundering)
- **Topological Sort**: Dependency resolution
- **Union-Find**: Connected components in transaction networks

### 4. Pricing & Calculations
- **DP**: Options pricing, portfolio optimization
- **Math**: Interest calculations, currency conversion
- **Precision**: Use BigDecimal for money

---

## Banking-Specific Problems
1. **Order Book Implementation** - TreeMap (Red-Black tree)
2. **Transaction Matching** - Two pointers
3. **Risk Calculation** - Sliding window
4. **Fraud Detection** - Graph algorithms
5. **Portfolio Optimization** - Knapsack DP
6. **Currency Exchange** - Coin change DP
7. **Payment Routing** - Dijkstra's algorithm
8. **Sanctions Screening** - Edit distance (fuzzy matching)
9. **Trade Execution** - Interval DP
10. **Real-time Analytics** - Two heaps (median)

---

## Key Concepts

### Financial Domain
- **Order Book**: Bid/ask prices, matching engine
- **Settlement**: T+2, T+3 cycles
- **Clearing**: Netting, reconciliation
- **Risk Metrics**: VaR, drawdown, Sharpe ratio
- **Compliance**: KYC, AML, OFAC sanctions

### Technical Requirements
- **Precision**: BigDecimal for money, never double
- **Concurrency**: ConcurrentHashMap, locks
- **Performance**: Low latency (<1ms for trading)
- **Reliability**: 99.99% uptime
- **Audit Trail**: Immutable logs

---

## Code Example: Order Book
```java
class OrderBook {
    TreeMap<Double, Queue<Order>> buyOrders = 
        new TreeMap<>(Collections.reverseOrder());
    TreeMap<Double, Queue<Order>> sellOrders = new TreeMap<>();
    
    public void addOrder(Order order) {
        TreeMap<Double, Queue<Order>> book = 
            order.side == Side.BUY ? buyOrders : sellOrders;
        book.computeIfAbsent(order.price, k -> new LinkedList<>())
            .offer(order);
        matchOrders();
    }
    
    public double getBestBid() {
        return buyOrders.isEmpty() ? 0 : buyOrders.firstKey();
    }
    
    public double getBestAsk() {
        return sellOrders.isEmpty() ? 0 : sellOrders.firstKey();
    }
}
```

---

## Interview Tips
- **Use BigDecimal**: Never use double for money
- **Handle Edge Cases**: Negative amounts, zero, overflow
- **Thread Safety**: Discuss concurrency if relevant
- **Explain Domain**: Show financial knowledge
- **Precision Matters**: Rounding, decimal places
- **Regulatory Compliance**: Mention if applicable

---

## Common Questions
1. "How would you detect money laundering in transaction networks?"
2. "Design a real-time risk calculation system."
3. "Implement an order matching engine."
4. "How do you ensure data consistency in distributed transactions?"
5. "Calculate portfolio VaR using historical simulation."

---

**Next**: [Interview Strategies](49-interview-strategies.md)
