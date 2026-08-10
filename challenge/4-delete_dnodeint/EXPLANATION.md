# `delete_dnodeint_at_index` — how it works

## The idea

A doubly linked list is a chain where each node holds two pointers:

```
        ┌──────┐      ┌──────┐      ┌──────┐
 NULL ←─┤ prev │  ←───┤ prev │  ←───┤ prev │
        │  n=0 │      │  n=1 │      │  n=2 │
        │ next ├───→  │ next ├───→  │ next ├──→ ...
        └──────┘      └──────┘      └──────┘
```

To delete a node you have to do two things: **make its neighbours point past it**, then
**free it**.

The function does that in three phases:

1. **Walk** to the node at position `index`.
2. **Check** you actually got there (otherwise the index was too big → return `-1`).
3. **Unlink and free**, with a special case for position 0 because the head node has no
   `prev`.

## The one weird trick in this code

Most implementations use a local cursor like `dlistint_t *current`. This code instead
**walks `*head` itself** — it moves the caller's head pointer down the list — and keeps a
backup in `saved_head` so it can put it back at the end.

That's why you see:

- `saved_head = *head;` — remember the real start of the list
- `*head = (*head)->next;` — this is *not* changing the list, it's just moving the cursor
- `*head = saved_head;` — put the head pointer back where it belongs

It works, but it's the main reason the code reads confusingly. And it's exactly why
position 0 needs its own branch: there, the cursor *is* the head, so the head pointer
genuinely has to change.

## Concrete run: `delete_dnodeint_at_index(&head, 5)`

Starting list, as built in `main.c`:

```
index:   0    1    2    3    4    5     6     7
        [0]↔[1]↔[2]↔[3]↔[4]↔[98]↔[402]↔[1024]
head ────┘
```

### Phase 1 — walk

`saved_head = [0]`, `p = 0`. The loop runs while `p < 5`:

| iteration | `*head` now points at | `p` |
| --------- | --------------------- | --- |
| start     | `[0]`                 | 0   |
| 1         | `[1]`                 | 1   |
| 2         | `[2]`                 | 2   |
| 3         | `[3]`                 | 3   |
| 4         | `[4]`                 | 4   |
| 5         | `[98]`                | 5   |

`p == 5`, loop stops. `*head` is now the cursor sitting on `[98]` — the node to delete.

### Phase 2 — check

`p != index` is false (5 == 5), so we continue. This guard catches the case where the list
ran out early, e.g. asking for index 20 on an 8-node list: the cursor hits NULL, `p` stops
below 20, so it restores `saved_head` and returns `-1`.

### Phase 3 — unlink

`index` isn't 0, so we take the `else` branch:

```c
(*head)->prev->next = (*head)->next;   /* [4]->next  now points to [402] */
...
(*head)->next->prev = (*head)->prev;   /* [402]->prev now points to [4]  */
```

Before:

```
[4] ──next──→ [98] ──next──→ [402]
[4] ←──prev── [98] ←──prev── [402]
```

After:

```
        ┌────────next────────┐
[4] ────┘      [98]          ↓ [402]
[4] ←───────────────prev─────┘
```

`[98]` is now orphaned — nothing in the list points to it — so `free()` reclaims its
memory, and `*head = saved_head` restores the head pointer to `[0]`. Result:

```
[0]↔[1]↔[2]↔[3]↔[4]↔[402]↔[1024]
```

## The other case: `delete_dnodeint_at_index(&head, 0)`

The walk loop never runs (`p < 0` is immediately false), so the cursor is still the head.
There's no `prev` node to patch, so instead:

```c
tmp = (*head)->next;   /* remember [1]                    */
free(*head);           /* destroy [0]                     */
*head = tmp;           /* caller's head now points at [1] */
if (tmp != NULL)
    tmp->prev = NULL;  /* [1] is the new first node       */
```

The `if (tmp != NULL)` matters for a one-element list: after deleting it, `head` becomes
NULL and there's nothing to fix up.

```
before:  head → [0]↔[1]↔[2]↔...
after:   head → [1]↔[2]↔...   with [1]->prev = NULL
```

That's also the branch every call in `main.c` except line 26 goes through.

## Notes on the bugs in this challenge

### 1. The original `else` branch never unlinked anything

The line originally read:

```c
(*head)->prev->prev = (*head)->prev;   /* N->prev->prev = N->prev */
```

That makes the *previous* node's `prev` point at **itself**, which:

1. corrupts the backward chain (self-loop), and
2. never unlinks the node from the forward chain — `N->prev->next` still points at it.

Since `print_dlistint` walks the list via `->next`, the freed node stays reachable and gets
printed after being `free()`d — a use-after-free that often *looks* fine because the
allocator hasn't reused the memory yet. The correct line is:

```c
(*head)->prev->next = (*head)->next;
```

### 2. Changing `main.c` line 26 to index 0 hides the bug, it doesn't fix it

Every call in `main.c` except line 26 uses index `0`. Turning that `5` into a `0` means the
`else` branch is **never executed once** in the whole program, so the broken line is simply
never reached. Keeping index `5` is what exercises it — which is why the test main was
written that way.

### 3. Use-after-free still present

```c
(*head)->prev->next = (*head)->next;
free(*head);
if ((*head)->next)                    /* <-- reading freed memory */
    (*head)->next->prev = (*head)->prev;
```

The unlinking must finish *before* the free:

```c
else
{
    tmp = *head;
    tmp->prev->next = tmp->next;
    if (tmp->next)
        tmp->next->prev = tmp->prev;
    free(tmp);
    *head = saved_head;
}
```

### 4. `index == length` dereferences NULL

If `index` equals the list length exactly, the loop leaves `*head == NULL` with
`p == index`, so the `p != index` guard passes and the `else` branch dereferences NULL.
Changing the loop guard to `while (p < index && (*head)->next != NULL)` — or checking
`*head == NULL` after the loop — closes that hole.
