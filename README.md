

📄 Real-Time-Leaderboard-Analysis.md

# Assignment 3 — Real-Time Leaderboard

## Problem Overview

We process a live stream of game score updates:

Input format:
userId, scoreDelta, timestamp

System must maintain:

- Top 100 players (global leaderboard)
- Per-minute rankings
- Incremental score updates
- Efficient memory usage

The solution is designed for real-time processing and technical evaluation of Java Streams API usage.

---

# System Design Overview

We maintain three main data structures:

```java
// Total score of each user
Map<String, Integer> totalScores;

// Per-minute scores
// minute -> (user -> score)
Map<String, Map<String, Integer>> perMinuteScores;

// Top 100 players (min-heap)
PriorityQueue<Player> topKHeap;


---

1️⃣ Efficient Aggregation

What Is Efficient Aggregation?

Efficient aggregation means:

Updating scores in constant time

Avoiding full recomputation

Avoiding re-sorting entire datasets on every update



---

How It Was Implemented

totalScores.merge(event.userId, event.scoreDelta, Integer::sum);

Why This Is Efficient

merge() performs an O(1) update.

We do not recompute totals from scratch.

Each incoming event only affects one user.


This ensures the system scales linearly with the number of events.


---

Why Not Recompute Every Time?

Bad approach:

totalScores.entrySet()
    .stream()
    .sorted(...)
    .limit(100)
    .collect(...)

This would:

Re-sort ALL users every time

Cost O(N log N)

Not scale for large user bases


Efficient aggregation avoids this entirely.


---

2️⃣ Incremental Updates

What Are Incremental Updates?

Instead of recalculating everything, we update only what changed.

For each event:

1. Update total score


2. Update per-minute score


3. Update top-K structure




---

Code Snippet

public void processEvent(ScoreEvent event) {

    // Update total score
    totalScores.merge(event.userId, event.scoreDelta, Integer::sum);

    // Update per-minute
    perMinuteScores
        .computeIfAbsent(minuteKey, k -> new HashMap<>())
        .merge(event.userId, event.scoreDelta, Integer::sum);

    // Update Top-K Heap
    updateTopK(event.userId);
}


---

Complexity Per Event

Map update → O(1)

Heap update → O(log K)


Since K = 100:

O(log 100) ≈ constant

This makes the system suitable for high-frequency real-time streams.


---

3️⃣ Bounded Memory Thinking

What Is Bounded Memory?

We never allow the leaderboard structure to grow unbounded.

Even if:

1 million users exist


We only store:

Top 100 users in heap


---

Implementation

PriorityQueue<Player> topKHeap =
    new PriorityQueue<>(Comparator.comparingInt(p -> p.score));

Why a Min-Heap?

Smallest score stays at top.

If a new score > smallest → replace it.

Heap size never exceeds 100.



---

Memory Guarantee

Memory used for leaderboard = O(K)

Where:

K = 100

Not dependent on total number of users.

This is critical in real-time systems.


---

4️⃣ Custom Comparator Usage

Custom comparator defines ranking logic.

Heap Comparator

Comparator.comparingInt(p -> p.score)

This ensures:

Heap orders by score ascending

Smallest score is removed first



---

Stream Sorting Comparator

.sorted((a, b) -> Integer.compare(b.score, a.score))

This ensures:

Highest score appears first

Proper leaderboard ordering


Custom comparators provide:

Explicit ranking logic

Flexibility for future changes

Clear ordering semantics



---

5️⃣ Where Streams API Should Be Used

Streams were used selectively.

Appropriate Usage

Example: Generating snapshot leaderboard

return topKHeap.stream()
        .sorted((a, b) -> Integer.compare(b.score, a.score))
        .collect(Collectors.toList());

Streams are ideal for:

Snapshot queries

Reporting

Transformations

Read-only views


They improve readability and reduce boilerplate.


---

6️⃣ Where Streams API Should NOT Be Used

Streams were intentionally NOT used in:

Core event processing

Heap maintenance

Score mutation logic


Reason:

Streams:

Allocate intermediate objects

Add lambda overhead

Do not improve algorithm complexity

Reduce control in hot loops


For real-time high-frequency workloads, traditional data structures are more predictable and efficient.


---

Engineering Evaluation of Streams API

Use Streams When:

Performing collection transformations

Generating reports

Creating filtered views

Batch analytics

Code readability matters


Avoid Streams When:

Inside high-frequency update loops

Managing bounded memory structures

Implementing performance-critical algorithms

Precise control over execution is required



---

Final Technical Verdict

Streams API is:

✔ Excellent for declarative data transformation
✔ Cleaner for reporting and snapshots
✔ Maintainable and readable

But:

❌ Not a replacement for core data structures
❌ Not automatically more efficient
❌ Not ideal for real-time mutation-heavy workloads

For this leaderboard system:

Core engine → HashMap + MinHeap

Snapshot layer → Streams API


This hybrid approach balances:

Performance

Memory control

Readability

Maintainability



---

Conclusion

The system demonstrates:

Efficient aggregation via incremental updates

Bounded memory via Top-K min-heap

Custom comparator-driven ranking

Selective, justified use of Streams API


The key engineering decision was not “use streams everywhere,” but:

> Use the right tool for the right layer of the system.



This approach is scalable, predictable, and production-ready for real-time workloads.
