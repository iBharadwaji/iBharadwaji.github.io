---
layout: page
title: "Reverse a singly linked list"
permalink: /interview-prep/coding/reverse-linked-list/
qsection: coding
tags: [linked-list, pointers, recursion, tail-recursion]
difficulty: easy
asked_by: [apple]
confidence: shaky
last_reviewed: 2026-09-07
---

## Question

Reverse a singly linked list. Give the iterative solution and the recursive solution, state time and space complexity, and make the empty list and the single-node list both work.

## Key idea

Rewiring a node backwards overwrites `curr->next`, which is the only pointer to the rest of the list, so it has to be saved first. Ship the iterative version: three pointers, `O(1)` space, and return `prev`, which ends up on the old tail.

```cpp
struct Node { int data; Node* next; };
```

## Answer

### Iterative

![Prev, curr and next pointers before and after one iteration of the reversal loop, with the four numbered steps](/interview-prep/coding/reverse-linked-list/assets/reverse-linked-list-iterative.svg)

```cpp
Node* reverse(Node* head) {
    Node* prev = nullptr;
    Node* curr = head;
    while (curr) {
        Node* next = curr->next;   // 1. save the rest
        curr->next = prev;         // 2. flip this link
        prev = curr;               // 3. advance prev
        curr = next;               // 4. advance curr
    }
    return prev;                   // old tail is the new head
}
```

The order of those four lines *is* the problem. Line 1 exists only because line 2 destroys the forward pointer. Mid-loop the list is genuinely severed between `prev` and `curr`, which is what forces the save.

Boundaries need no special-casing. An empty list skips the loop and returns `nullptr`. A single node runs once and returns itself. The old head becomes the tail with a null `next`, because `prev` starts null.

### Recursive, classic

Rewires while unwinding. This is the version most interviewers have in mind.

![Recursion descending to the tail, then one frame rewiring on the way back up because head's own pointer was never modified](/interview-prep/coding/reverse-linked-list/assets/reverse-linked-list-recursive.svg)

```cpp
Node* reverse(Node* head) {
    if (!head || !head->next) return head;   // 0 or 1 node
    Node* newHead = reverse(head->next);
    head->next->next = head;   // successor points back at me
    head->next = nullptr;      // I become the tail
    return newHead;
}
```

The load-bearing insight: after the call returns, the sublist from `head->next` onward is fully reversed and the node originally at `head->next` is now its **tail**. Because the recursion rewires only downstream and never touches `head`'s own pointer, `head->next` is still a valid handle on that tail — which is what makes the stitch `O(1)`.

Note `head == nullptr` is reachable only on the very first call; inside the recursion `head->next == nullptr` catches the last node one step early. It is purely an empty-list guard.

### Recursive, accumulator

No diagram, because the mechanism is **identical to the iterative version** — that equivalence is the thing worth remembering. Each call saves `next`, rewires backwards, and advances both pointers; the only difference is that the advance is a function call instead of a loop increment.

```cpp
Node* reverse(Node* curr, Node* prev = nullptr) {
    if (!curr) return prev;
    Node* next = curr->next;
    curr->next = prev;
    return reverse(next, curr);
}
```

The base case `if (!curr) return prev;` does three jobs at once: it terminates, it handles the empty list without dereferencing, and `prev` *is* the answer, because falling off the end leaves it on the last node. Recursing one step onto the null pointer is what buys all three.

This one is genuinely tail-recursive — the call is the last thing the function does — so GCC and Clang at `-O2` collapse it into the loop above.

## Cost

| Version | Time | Space | Tail position |
|---|---|---|---|
| Iterative | `O(n)` | `O(1)` | — |
| Accumulator recursive | `O(n)` | `O(n)`, `O(1)` if TCO | yes |
| Classic recursive | `O(n)` | `O(n)` | no |

Ship the iterative one. Tail-call elimination is not guaranteed by the standard, so claim `O(n)` stack and offer `O(1)` only as a compiler optimisation.

## Gotchas

- Save `curr->next` **before** `curr->next = prev`. Reverse those and everything past `curr` is unreachable.
- `prev` must start `nullptr`, not `head`. Otherwise the old head keeps pointing at the second node while the second points back at it, creating a two-node cycle.
- In the classic recursive form, `head->next = nullptr` is strictly needed only in the outermost frame — inner frames have it overwritten by their parent. Doing it uniformly is correct and simpler. Dropping it entirely gives you the cycle above.
- The classic recursive version is **not** tail-recursive: three statements follow the call, so the frame must survive it and `O(n)` stack is unavoidable.
- Assert the old head's `next` is null afterwards. The cycle bug makes a print-traversal hang forever; Floyd's tortoise and hare catches it.
- Test empty, one node, two nodes (the smallest case where order changes), and one odd and one even length.

## Follow-ups

- **Reverse in groups of k.** Confirm `k` nodes exist before rewiring, since you cannot back out mid-flip. A dummy node ahead of the head keeps the stitching uniform, because each group's old head becomes its new tail.
- **Reverse a sublist between positions m and n.** Same dummy-node technique.
- **Palindrome check.** Find the middle with slow/fast pointers, reverse the second half, compare.
- **Detect a cycle.** Floyd's tortoise and hare — the assertion that catches the `prev` initialisation bug above.
- **Doubly linked list.** Swap `next` and `prev` on every node and swap the list's head and tail.
