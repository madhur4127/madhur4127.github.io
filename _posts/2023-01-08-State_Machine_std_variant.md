---
title: Powerful state machine management with std::variant
date: 2023-01-08 00:00:00 +0100
categories: [C++]
tags: [c++]     # TAG names should always be lowercase
---

# Powerful state machine management with std::variant

Canonically state machines are handled by enums in C++. For e.g. in a (simple) trading system, an order can have these simple states:

```c++
enum class OrderState {
  InsertPending, // Insert request is inflight, waiting on exchange
  Inserted, // Exchange acknowledged the insert
  CancelPending, // Cancel inflight
  Finished // traded / cancelled / rejected by exchange
};
```

## Motivating example
Suppose you want to track the round trip response time for an insert i.e. Order Insert Req Sent ->  Exchange Received the request -> Exchange's Acknowledgement Sent -> Exchange Acknowledgement received.

This means you need to have your `Order` struct somewhat like:

```c++
using time_point = std::chrono::high_resolution_clock::time_point;

struct Order {
   // ...
   OrderState state_;
   time_point request_sent_at_;
};
```

It poses a problem that if the order is in `Inserted` state, there's no inflight request so having a `time_point` is invalid. It can be fixed by `std::optional`:

```c++
struct Order {
   // ...
   OrderState state_;
   std::optional<time_point> req_sent_at_;
};
```
## Alternative solution

You can combine enum and the data associated with each state into separate structs:

```c++

struct InsertPending {
    time_point request_sent_at_;
};
struct Inserted {};
struct CancelPending {
    time_point request_sent_at_;
};
struct Finished {};

struct Order {
    // ...
    std::variant<InsertPending, Inserted, CancelPending, Finished> state_;
};
```

It looks complicated but the benefit of this approach is that you don't have to clutter the `Order` with details that is specific to each state.

For e.g. what if you want to store the reason why an order is in finished state? Using simple enum style:

```c++
struct Order {
  // ...
  OrderState state_;
  time_point req_sent_at_;

  enum class FinalizedReason { NotFinalized, PartiallyTraded, FullyTraded, Cancelled };
  FinalizedReason reason_;
};
```

Problems:
1. Order struct is now wasting space as `req_sent_at_` and `reason_` are mutually exclusive.
2. Need to keep two enums in sync i.e. when `OrderState` is not `Finalized`, `reason_` needs to be in `NotFinalized` state. This can be source of bugs and these implicit assumptions makes code harder to maintain.

Using the alternative solution, we only need to change `Finalized` class:

```c++
struct Finalized {
  enum class Reason { PartiallyTraded, FullyTraded, Cancelled };
  Reason reason_;
};
```
`Order` is unchanged and is same as before:

```c++
struct Order {
  //...
  std::variant<InsertPending, Inserted, CancelPending, Finished> state_;
};
```