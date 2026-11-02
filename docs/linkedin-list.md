# Linked Lists


```java
class ListNode {
    int val;
    ListNode next;

    ListNode(int val) {
        this.val = val;
        this.next = null;
    }
}
```
## Common linked-list problems

1. **Reverse a linked list** — Reverse the links in place, iteratively or recursively.
2. **Detect a cycle** — Determine whether a list loops back to an earlier node, commonly using slow and fast pointers.
3. **Merge two sorted lists** — Combine two sorted lists into one sorted list.
4. **Remove the Nth node from the end** — Find and remove a node in one pass using two pointers.
5. **Find the intersection of two linked lists** — Determine whether two lists share a node, often by aligning traversal lengths.
6. **Reorder List** — Rearrange the list by alternating nodes from its beginning and end.
7. **Copy List With Random Pointer** — Deep-copy a list where each node also points to any node in the list or `null`.
8. **Add Two Numbers** — Add numbers stored as reverse-order linked lists of digits.
9. **Find the Duplicate Number** — Find the duplicate in an array of `n + 1` values from `1` to `n`.
10. **LRU Cache** — Support least-recently-used eviction with constant-time get and put operations.

## 1. Reverse a linked list

### Mental model and pseudocode

Reverse each link as you walk forward, keeping track of the previous node.

```text
previous = null
current = head
while current exists:
    save current.next
    point current.next to previous
    move previous to current
    move current to saved next
return previous
```

### Code

```java
ListNode reverseList(ListNode head) {
    ListNode previous = null;
    ListNode current = head;
    while (current != null) {
        ListNode next = current.next;
        current.next = previous;
        previous = current;
        current = next;
    }
    return previous;
}
```

## 2. Detect a cycle

### Mental model and pseudocode

Imagine two runners on the list: a slow runner moves one node at a time, and a fast runner moves two. If the list loops, the fast runner eventually catches the slow runner. If the fast runner reaches the end, there is no cycle.

```text
slow = head
fast = head
while fast and fast.next exist:
    move slow forward by one node
    move fast forward by two nodes
    if slow and fast are the same node:
        return true
return false
```

### Code

```java
boolean hasCycle(ListNode head) {
    ListNode slow = head;
    ListNode fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;
    }
    return false;
}
```

## 3. Merge two sorted lists

### Mental model and pseudocode

Repeatedly attach the smaller current node to the result, then attach whichever list remains.

```text
use a dummy node and a tail pointer
while both lists exist:
    attach the smaller current node to tail
    advance that list and tail
attach the remaining list
return dummy.next
```

### Code

```java
ListNode mergeTwoLists(ListNode list1, ListNode list2) {
    ListNode dummy = new ListNode(0);
    ListNode tail = dummy;
    while (list1 != null && list2 != null) {
        if (list1.val <= list2.val) {
            tail.next = list1;
            list1 = list1.next;
        } else {
            tail.next = list2;
            list2 = list2.next;
        }
        tail = tail.next;
    }
    tail.next = list1 != null ? list1 : list2;
    return dummy.next;
}
```

## 4. Remove the Nth node from the end

### Mental model and pseudocode

Keep two pointers `n` nodes apart. When the front pointer reaches the end, the back pointer is just before the node to remove.

```text
create a dummy node before head
move fast forward n nodes from dummy
move fast and slow forward together until fast is last
skip slow.next
return dummy.next
```

### Code

```java
ListNode removeNthFromEnd(ListNode head, int n) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode fast = dummy;
    ListNode slow = dummy;
    for (int i = 0; i < n; i++) {
        fast = fast.next;
    }
    while (fast.next != null) {
        fast = fast.next;
        slow = slow.next;
    }
    slow.next = slow.next.next;
    return dummy.next;
}
```

## 5. Find the intersection of two linked lists

### Mental model and pseudocode

Walk both pointers through both lists. Switching to the other list after reaching the end makes their total path lengths equal. They meet at the shared node, or both reach the end if there is no intersection.

```text
pointerA = headA
pointerB = headB
while pointerA and pointerB are different:
    move pointerA one step, or to headB if at the end
    move pointerB one step, or to headA if at the end
return pointerA
```

### Code

```java
ListNode getIntersectionNode(ListNode headA, ListNode headB) {
    ListNode pointerA = headA;
    ListNode pointerB = headB;
    while (pointerA != pointerB) {
        pointerA = pointerA == null ? headB : pointerA.next;
        pointerB = pointerB == null ? headA : pointerB.next;
    }
    return pointerA;
}
```

## 6. Reorder List

### Mental model and pseudocode

Split the list in half, reverse the second half, then weave the two halves together.

```text
find the middle
reverse the second half
alternate nodes from each half
```

### Code

```java
void reorderList(ListNode head) {
    if (head == null || head.next == null) return;

    ListNode slow = head;
    ListNode fast = head;
    while (fast.next != null && fast.next.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }

    ListNode second = reverseList(slow.next);
    slow.next = null;
    ListNode first = head;
    while (second != null) {
        ListNode firstNext = first.next;
        ListNode secondNext = second.next;
        first.next = second;
        second.next = firstNext;
        first = firstNext;
        second = secondNext;
    }
}
```

## 7. Copy List With Random Pointer

### Mental model and pseudocode

Make one copy per original node, then use a map to connect each copy's `next` and `random` pointers.

```text
create a copy of every node and store original -> copy
for each original node:
    connect its copy's next and random using the map
return the copy of the head
```

### Code

```java
import java.util.HashMap;
import java.util.Map;

class RandomNode {
    int val;
    RandomNode next;
    RandomNode random;

    RandomNode(int val) {
        this.val = val;
    }
}

RandomNode copyRandomList(RandomNode head) {
    Map<RandomNode, RandomNode> copies = new HashMap<>();
    for (RandomNode node = head; node != null; node = node.next) {
        copies.put(node, new RandomNode(node.val));
    }
    for (RandomNode node = head; node != null; node = node.next) {
        RandomNode copy = copies.get(node);
        copy.next = copies.get(node.next);
        copy.random = copies.get(node.random);
    }
    return copies.get(head);
}
```

## 8. Add Two Numbers

### Mental model and pseudocode

Add matching digits and carry just like elementary addition. Continue while either list or a carry remains.

```text
use a dummy node and carry = 0
while either list has a digit or carry is nonzero:
    sum the digits and carry
    append sum % 10
    set carry = sum / 10
return dummy.next
```

### Code

```java
ListNode addTwoNumbers(ListNode list1, ListNode list2) {
    ListNode dummy = new ListNode(0);
    ListNode tail = dummy;
    int carry = 0;

    while (list1 != null || list2 != null || carry != 0) {
        int sum = carry;
        if (list1 != null) {
            sum += list1.val;
            list1 = list1.next;
        }
        if (list2 != null) {
            sum += list2.val;
            list2 = list2.next;
        }
        tail.next = new ListNode(sum % 10);
        carry = sum / 10;
        tail = tail.next;
    }
    return dummy.next;
}
```

## 9. Find the Duplicate Number

### Mental model and pseudocode

Treat each value as a pointer to the next array index. The duplicate creates a cycle; Floyd's slow and fast pointers find its entrance.

```text
find a meeting point using slow = nums[slow], fast = nums[nums[fast]]
start one pointer at index 0
move both one step at a time
their meeting index is the duplicate
```

### Code

```java
int findDuplicate(int[] nums) {
    int slow = nums[0];
    int fast = nums[0];
    do {
        slow = nums[slow];
        fast = nums[nums[fast]];
    } while (slow != fast);

    slow = nums[0];
    while (slow != fast) {
        slow = nums[slow];
        fast = nums[fast];
    }
    return slow;
}
```

## 10. LRU Cache

### Mental model and pseudocode

Keep entries in access order: each get or put makes that key most recently used. When over capacity, remove the oldest entry.

```text
get(key): return its value and mark it most recently used
put(key, value): store/update it and mark it most recently used
if over capacity: remove the least recently used entry
```

### Code

```java
import java.util.LinkedHashMap;
import java.util.Map;

class LRUCache {
    private final int capacity;
    private final LinkedHashMap<Integer, Integer> cache;

    LRUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new LinkedHashMap<Integer, Integer>(capacity, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
                return size() > LRUCache.this.capacity;
            }
        };
    }

    int get(int key) {
        return cache.getOrDefault(key, -1);
    }

    void put(int key, int value) {
        cache.put(key, value);
    }
}
```
