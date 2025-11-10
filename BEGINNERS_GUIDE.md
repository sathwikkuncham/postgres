# PostgreSQL for Beginners: Understanding a Complex Codebase from First Principles

## Introduction: What Makes Managing Large Codebases Hard?

Before we dive into PostgreSQL specifically, let's understand the fundamental problem.

### The Problem: Complexity Grows Exponentially

Imagine you're building a small program:
- 100 lines of code? Easy to understand. You can read it all in one sitting.
- 1,000 lines? Still manageable. Maybe a few files, some functions.
- 10,000 lines? Now you need organization. Files in folders, clear naming.
- 100,000 lines? You need architecture. Systems, subsystems, interfaces.
- **2,000,000+ lines?** (PostgreSQL's size) - You need **patterns and discipline**.

Without patterns, here's what happens:
- **Change one thing, break everything** - tight coupling
- **Memory leaks everywhere** - forgot to free resources
- **Code duplication** - same logic copied 100 times
- **No one understands the whole system** - knowledge silos
- **Can't add features** - too fragile to modify

PostgreSQL solved these problems using **design patterns**. Let's learn them from scratch.

---

## Part 1: Foundational Concepts (Building Blocks)

Before we look at PostgreSQL, we need to understand the building blocks. I'll explain each concept in isolation, then show how they combine.

---

### Concept 1: Structs in C (Grouping Related Data)

**What is a struct?**

A struct is C's way of grouping related data together. Think of it like a record or a form.

**Example - Without structs:**
```c
// Messy - lots of separate variables
char person_name[50];
int person_age;
float person_height;

char another_person_name[50];
int another_person_age;
float another_person_height;
```

**Example - With structs:**
```c
// Clean - data grouped together
struct Person {
    char name[50];
    int age;
    float height;
};

// Now we can create people easily
struct Person alice;
alice.age = 30;

struct Person bob;
bob.age = 25;
```

**Why this matters:** PostgreSQL has thousands of different types of data structures. Structs let them organize this data cleanly.

---

### Concept 2: Pointers (References to Data)

**What is a pointer?**

A pointer is a variable that stores the **address** of another variable. Think of it like a house address - it tells you where to find something, rather than being the thing itself.

**Simple analogy:**
```
Regular variable:  A box containing an apple
Pointer:          A piece of paper with directions to a box containing an apple
```

**Code example:**
```c
int age = 30;           // A box with the number 30
int *ptr = &age;        // A pointer storing the address of 'age'

printf("%d", age);      // Prints 30 (direct access)
printf("%d", *ptr);     // Prints 30 (indirect access through pointer)
```

**Why pointers matter:**
1. **Pass large data efficiently** - Instead of copying entire structures, pass their address
2. **Share data** - Multiple parts of code can access the same data
3. **Dynamic memory** - Allocate memory at runtime (we'll cover this next)

**Common confusion:** The `*` symbol means different things in different contexts:
```c
int *ptr;          // Declaration: "ptr is a pointer to an int"
int value = *ptr;  // Dereferencing: "get the value that ptr points to"
```

---

### Concept 3: Dynamic Memory Allocation (Creating Data at Runtime)

**The problem with regular variables:**

```c
int numbers[10];  // Fixed size - always 10 integers, decided at compile time
```

What if you don't know how many you need until the program runs?

**Solution: Dynamic allocation**

```c
// At runtime, decide size
int count = get_user_input();  // User says: "I need 50 numbers"

// Allocate exactly that much memory
int *numbers = malloc(count * sizeof(int));

// Use it
numbers[0] = 42;
numbers[1] = 100;

// When done, FREE the memory
free(numbers);
```

**Key points:**
- `malloc` = **m**emory **alloc**ation - get memory from the operating system
- Returns a pointer to the allocated memory
- **You must call `free()`** when done, or you get a **memory leak**
- Memory leak = allocated memory that's never freed (program uses more and more RAM)

**The Big Problem:**
```c
void process_data() {
    int *data = malloc(1000);

    // ... do work ...

    if (error_happened) {
        return;  // BUG! Forgot to free(data) - MEMORY LEAK!
    }

    free(data);  // Only freed if no error
}
```

In large programs, tracking all allocations and making sure they're freed is **extremely difficult**.

**PostgreSQL's solution:** Memory Contexts (we'll cover this in Part 2)

---

### Concept 4: Function Pointers (Storing References to Functions)

Just like you can store addresses of data, you can store addresses of **functions**.

**Why would you want this?**

Imagine you have different sorting algorithms:
```c
void bubble_sort(int *arr, int size) { /* ... */ }
void quick_sort(int *arr, int size) { /* ... */ }
void merge_sort(int *arr, int size) { /* ... */ }
```

Instead of this:
```c
if (user_choice == 1)
    bubble_sort(data, size);
else if (user_choice == 2)
    quick_sort(data, size);
else if (user_choice == 3)
    merge_sort(data, size);
```

You can do this:
```c
// Declare a function pointer type
typedef void (*SortFunction)(int *arr, int size);

// Choose which function to use
SortFunction sorter;
if (user_choice == 1)
    sorter = bubble_sort;
else if (user_choice == 2)
    sorter = quick_sort;
else
    sorter = merge_sort;

// Call the chosen function
sorter(data, size);
```

**Why this matters for PostgreSQL:**
- Different index types (B-tree, Hash, GIN) all need different implementations
- Different memory allocators need different strategies
- Function pointers let you **choose behavior at runtime**

---

### Concept 5: Type Casting (Telling the Compiler to Treat Data Differently)

**The problem:**

C is strongly typed. The compiler enforces types:
```c
int x = 5;
float y = x;  // Error! Can't assign int to float without explicit conversion
```

**Solution: Casting**
```c
int x = 5;
float y = (float)x;  // OK! We explicitly told compiler to convert
```

**Pointer casting (more dangerous):**
```c
void *generic_ptr = malloc(100);  // void* = "pointer to unknown type"
int *int_ptr = (int *)generic_ptr;  // "Treat this as pointer to int"
```

**Why this matters:** PostgreSQL uses "base types" that get cast to specific types:
```c
// Generic node
struct Node {
    int type;  // Identifies what kind of node this really is
};

// Specific node types
struct SelectStatement {
    int type;  // Must be first!
    char *table_name;
    // ... more fields
};

struct InsertStatement {
    int type;  // Must be first!
    int num_values;
    // ... more fields
};

// Later, you can do:
struct Node *node = get_next_statement();
if (node->type == SELECT_TYPE) {
    struct SelectStatement *select = (struct SelectStatement *)node;
    // Now use select->table_name
}
```

This pattern is called a **tagged union** (we'll explore this more in Part 2).

---

## Part 2: Design Patterns (Solutions to Common Problems)

Now that we understand the C building blocks, let's learn the design patterns PostgreSQL uses.

---

### Pattern 1: Tagged Union (One Type That Can Be Many Things)

**The Problem:**

You're parsing SQL. You might encounter:
- `SELECT` statements
- `INSERT` statements
- `UPDATE` statements
- `DELETE` statements
- `CREATE TABLE` statements
- ... dozens more

You need to:
1. Store all these different types
2. Know which type you're dealing with
3. Access type-specific data

**Bad Solution #1: Separate variables**
```c
struct SelectStmt *select = NULL;
struct InsertStmt *insert = NULL;
struct UpdateStmt *update = NULL;

// Messy! Have to check all of them
if (select != NULL)
    process_select(select);
else if (insert != NULL)
    process_insert(insert);
// ...
```

**Bad Solution #2: Giant struct with all fields**
```c
struct Statement {
    char *table_name;        // Only for SELECT/INSERT/UPDATE
    int num_columns;         // Only for SELECT/CREATE
    int num_values;          // Only for INSERT
    bool is_temporary;       // Only for CREATE TABLE
    // Hundreds of fields, most unused for any given statement
};
```
This wastes memory - most fields are unused for any specific statement type.

**Good Solution: Tagged Union**

```c
// Step 1: Define a "tag" - an enum listing all possible types
typedef enum NodeTag {
    T_SelectStmt,
    T_InsertStmt,
    T_UpdateStmt,
    T_DeleteStmt,
    // ... more types
} NodeTag;

// Step 2: Base structure that ALL types share
struct Node {
    NodeTag type;  // The "tag" - MUST be first field
};

// Step 3: Specific structures (all start with NodeTag)
struct SelectStmt {
    NodeTag type;           // MUST be first!
    char *table_name;
    int num_columns;
    // SELECT-specific fields
};

struct InsertStmt {
    NodeTag type;           // MUST be first!
    char *table_name;
    int num_values;
    // INSERT-specific fields
};

// Step 4: Use it
struct Node *stmt = parse_sql(query);

// Check the tag to see what we actually have
switch (stmt->type) {
    case T_SelectStmt:
        // Safe to cast because we checked the tag
        struct SelectStmt *select = (struct SelectStmt *)stmt;
        printf("Selecting from: %s\n", select->table_name);
        break;

    case T_InsertStmt:
        struct InsertStmt *insert = (struct InsertStmt *)stmt;
        printf("Inserting %d values\n", insert->num_values);
        break;

    // ... handle other types
}
```

**Key insight:**
- The `type` field at the start tells you what the structure **really** is
- You can pass around `Node *` pointers everywhere (generic)
- When you need specifics, check the tag and cast

**Why PostgreSQL uses this:**
- **200+ different node types** in the system (statements, expressions, plans, etc.)
- All can be passed around as `Node *`
- Switch statements dispatch based on tag
- Type-safe (the tag prevents wrong casts)

---

### Pattern 2: Method Tables (Polymorphism Without Classes)

**The Problem:**

You have multiple implementations of the same concept. For example, sorting:
- Bubble sort
- Quick sort
- Merge sort

They all do the same job (sort an array) but with different algorithms.

In object-oriented languages (Java, C++), you'd use:
```java
interface Sorter {
    void sort(int[] array);
}

class BubbleSort implements Sorter { /* ... */ }
class QuickSort implements Sorter { /* ... */ }
```

**But C doesn't have classes or interfaces!**

**Solution: Method Tables (aka Virtual Function Tables)**

A method table is a struct containing function pointers:

```c
// Step 1: Define the "interface" as a struct of function pointers
typedef struct SorterMethods {
    void (*sort)(int *array, int size);
    void (*display_stats)(void);
    const char *algorithm_name;
} SorterMethods;

// Step 2: Implement different versions
void bubble_sort_impl(int *array, int size) {
    // Bubble sort implementation
}

void bubble_stats(void) {
    printf("Bubble sort: O(n^2)\n");
}

SorterMethods bubble_sorter = {
    .sort = bubble_sort_impl,
    .display_stats = bubble_stats,
    .algorithm_name = "Bubble Sort"
};

void quick_sort_impl(int *array, int size) {
    // Quick sort implementation
}

void quick_stats(void) {
    printf("Quick sort: O(n log n)\n");
}

SorterMethods quick_sorter = {
    .sort = quick_sort_impl,
    .display_stats = quick_stats,
    .algorithm_name = "Quick Sort"
};

// Step 3: Use polymorphically
void sort_data(int *array, int size, SorterMethods *methods) {
    printf("Using: %s\n", methods->algorithm_name);
    methods->sort(array, size);
    methods->display_stats();
}

// Step 4: Call with any implementation
int data[] = {5, 2, 8, 1, 9};
sort_data(data, 5, &bubble_sorter);  // Uses bubble sort
sort_data(data, 5, &quick_sorter);   // Uses quick sort
```

**Benefits:**
- Add new implementations without changing existing code
- Choose implementation at runtime
- Code using the methods doesn't need to know which implementation it's calling

**PostgreSQL examples:**
1. **Memory allocators** - Different strategies (we'll explore next)
2. **Index types** - B-tree, Hash, GIN all implement same interface
3. **Table storage** - Different storage engines all implement same interface

---

### Pattern 3: Hierarchical Resource Management (Parent-Child Cleanup)

**The Problem: Memory Leak Hell**

```c
void complex_operation() {
    char *buffer1 = malloc(1000);
    int *data = malloc(500 * sizeof(int));
    char *temp = malloc(200);

    if (step1_fails()) {
        free(buffer1);
        return;  // BUG! Forgot to free data and temp
    }

    if (step2_fails()) {
        free(buffer1);
        free(data);
        return;  // BUG! Forgot to free temp
    }

    // Success path
    free(buffer1);
    free(data);
    free(temp);
}
```

As functions get more complex, tracking every allocation is impossible. One forgotten `free()` = memory leak.

**Solution: Hierarchical Resource Management**

**Analogy: Organizing your house**

Bad approach:
```
Throw everything on the living room floor.
When cleaning: Pick up each item individually.
If you miss one: Lost forever under the couch.
```

Good approach:
```
Living Room
  ├── Kitchen Stuff Box
  │     ├── Plates
  │     └── Utensils
  ├── Bedroom Stuff Box
  │     ├── Clothes
  │     └── Books
  └── Bathroom Stuff Box
        ├── Towels
        └── Toiletries

When cleaning: Just throw away entire boxes.
Everything inside automatically cleaned up!
```

**In code:**

```c
// Context = a "container" for allocations
typedef struct MemoryContext {
    char *name;
    struct MemoryContext *parent;
    struct MemoryContext *first_child;
    void *allocated_blocks;  // All allocations in this context
} MemoryContext;

// Root context (never deleted)
MemoryContext *TopContext;

// Create a context for a specific operation
MemoryContext *query_context = create_context("Query", TopContext);

// Switch to that context
set_current_context(query_context);

// All allocations now go in query_context
char *buffer = palloc(1000);    // Goes in query_context
int *data = palloc(500);        // Goes in query_context
char *temp = palloc(200);       // Goes in query_context

// Later: Clean up everything at once
delete_context(query_context);  // ALL allocations freed automatically!
```

**Key insight:**
- Allocations are grouped by **lifetime**
- Delete the context = free everything in it (and child contexts)
- **No way to forget to free something** - it's automatic

**PostgreSQL's hierarchy:**
```
TopMemoryContext (permanent - never deleted)
  ├── CacheMemoryContext (system caches)
  │
  ├── TransactionContext (deleted at commit/rollback)
  │     ├── Portal1Context (one SQL query)
  │     │     ├── ExecutorContext (query execution)
  │     │     │     └── PerTupleContext (reset for each row)
  │     │     │
  │     │     └── PlannerContext (query planning)
  │     │
  │     └── Portal2Context (another query)
  │           └── ...
```

When a transaction commits:
1. Delete `TransactionContext`
2. All portals automatically deleted
3. All executor contexts automatically deleted
4. All per-tuple contexts automatically deleted
5. **No memory leaks possible!**

---

### Pattern 4: Code Generation (Let the Computer Write Repetitive Code)

**The Problem: Repetitive Code**

You have 200 different types of nodes (SelectStmt, InsertStmt, etc.). For each one, you need:
- A function to copy it: `copy_SelectStmt()`, `copy_InsertStmt()`, ...
- A function to compare it: `equal_SelectStmt()`, `equal_InsertStmt()`, ...
- A function to serialize it: `serialize_SelectStmt()`, ...

Writing these **by hand for 200+ types = 800+ functions**.

**Worse:** Every time you add a field to a struct, you must update 4 functions. **Extremely error-prone.**

**Solution: Code Generation**

Write a program that **reads your struct definitions** and **automatically generates** all the repetitive code.

**PostgreSQL's approach:**

**Step 1:** Annotate your structs:
```c
typedef struct SelectStmt {
    NodeTag type;

    char *table_name;

    List *columns pg_node_attr(copy_as(copyObjectImpl));

    int64 query_id pg_node_attr(equal_ignore);  // Don't compare this field
} SelectStmt;
```

**Step 2:** Perl script (`gen_node_support.pl`) reads these definitions and generates:

```c
// Generated automatically in copyfuncs.c
SelectStmt *copy_SelectStmt(SelectStmt *from) {
    SelectStmt *newnode = makeNode(SelectStmt);

    newnode->type = from->type;
    COPY_STRING_FIELD(table_name);
    COPY_NODE_FIELD(columns);
    COPY_SCALAR_FIELD(query_id);

    return newnode;
}

// Generated automatically in equalfuncs.c
bool equal_SelectStmt(SelectStmt *a, SelectStmt *b) {
    COMPARE_STRING_FIELD(table_name);
    COMPARE_NODE_FIELD(columns);
    // query_id skipped because of equal_ignore attribute

    return true;
}
```

**Benefits:**
- Write struct definition once
- Get copy/equal/serialize functions for free
- Add a field? Functions automatically updated
- Consistent implementation across all types
- No human errors

---

## Part 3: How PostgreSQL Combines These Patterns

Now let's see how these patterns work together to manage complexity.

---

### The Node System: Tagged Unions + Code Generation

**Remember:**
- Tagged Union = base struct with a type tag, specific structs extend it
- Code Generation = automatically create functions for all types

**PostgreSQL combines them:**

```c
// nodes.h - Base type
typedef enum NodeTag {
    T_SelectStmt,
    T_InsertStmt,
    // ... 200+ types
} NodeTag;

typedef struct Node {
    NodeTag type;
} Node;

// parsenodes.h - Parse tree nodes
typedef struct SelectStmt {
    NodeTag type;
    char *table_name;
    List *target_list;
} SelectStmt;

// plannodes.h - Plan tree nodes
typedef struct SeqScan {
    NodeTag type;
    int scan_relid;
    List *qual;
} SeqScan;

// execnodes.h - Execution tree nodes
typedef struct SeqScanState {
    NodeTag type;
    SeqScan *plan;
    Relation relation;
    // ... runtime state
} SeqScanState;
```

**Code generation creates:**
- `copyObject()` - works for **any** node type
- `equal()` - works for **any** node type
- Serialization - works for **any** node type

**Usage:**
```c
// Generic pointer
Node *stmt = parse_sql("SELECT * FROM users");

// Copy works for any node
Node *copy = copyObject(stmt);

// Compare works for any node
bool same = equal(stmt, copy);  // true

// Check what it actually is
if (stmt->type == T_SelectStmt) {
    SelectStmt *select = (SelectStmt *)stmt;
    printf("Table: %s\n", select->table_name);
}
```

---

### Memory Management: Hierarchical Contexts + Method Tables

**Remember:**
- Hierarchical contexts = group allocations by lifetime
- Method tables = different implementations of same interface

**PostgreSQL combines them:**

**Different allocation strategies for different use cases:**

```c
// Method table for memory allocators
typedef struct MemoryContextMethods {
    void *(*alloc)(MemoryContext context, Size size);
    void (*free_p)(void *pointer);
    void (*reset)(MemoryContext context);
    // ...
} MemoryContextMethods;

// AllocSet - general purpose (default)
MemoryContextMethods AllocSetMethods = {
    .alloc = AllocSetAlloc,
    .free_p = AllocSetFree,
    .reset = AllocSetReset,
    // ...
};

// Slab - for fixed-size allocations (e.g., all nodes same size)
MemoryContextMethods SlabMethods = {
    .alloc = SlabAlloc,
    .free_p = SlabFree,
    .reset = SlabReset,
    // ...
};

// Generation - for FIFO allocations (allocate in order, free in order)
MemoryContextMethods GenerationMethods = {
    .alloc = GenerationAlloc,
    .free_p = GenerationFree,
    .reset = GenerationReset,
    // ...
};
```

**Creating contexts with different strategies:**

```c
// Create a general-purpose context
MemoryContext query_ctx = AllocSetContextCreate(
    TopMemoryContext,    // parent
    "Query Context",     // name
    &AllocSetMethods     // use AllocSet strategy
);

// Create a context for fixed-size node allocations
MemoryContext node_ctx = SlabContextCreate(
    query_ctx,           // parent (child of query_ctx)
    "Node Context",
    &SlabMethods,        // use Slab strategy
    sizeof(SelectStmt)   // all allocations this size
);
```

**Key insight:**
- Different parts of the system have different allocation patterns
- Choose the right strategy for each use case
- All use the same interface (`palloc`/`pfree`)
- User code doesn't need to know which strategy is used

---

### Query Processing: All Patterns Combined

Let's trace a query through the system and see all patterns in action:

**User runs:** `SELECT name FROM users WHERE age > 21`

#### Stage 1: Parsing (Tagged Unions + Memory Contexts)

```c
// Create a context for parsing
MemoryContext parse_ctx = create_context("Parse", TopMemoryContext);
switch_to_context(parse_ctx);

// Parser creates SelectStmt node
SelectStmt *stmt = makeNode(SelectStmt);  // Tagged union
stmt->type = T_SelectStmt;
stmt->table_name = pstrdup("users");      // Allocated in parse_ctx
stmt->where_clause = /* parse WHERE clause */;

// Return the parse tree (still in parse_ctx memory)
return (Node *)stmt;
```

#### Stage 2: Planning (Tagged Unions + Method Tables + Memory Contexts)

```c
// Create a context for planning
MemoryContext plan_ctx = create_context("Planner", query_ctx);
switch_to_context(plan_ctx);

// Planner creates plan nodes
SeqScan *plan = makeNode(SeqScan);  // Tagged union
plan->type = T_SeqScan;
plan->scan_relid = /* table ID */;

// Cost estimation uses method tables
// Different index types provide different cost functions
IndexAmRoutine *btree_methods = get_index_methods(BTREE_AM);
Cost index_cost = btree_methods->amcostestimate(/* params */);

// Compare costs and choose best plan
// ...

return (Node *)plan;
```

#### Stage 3: Execution (Tagged Unions + Method Tables + Hierarchical Contexts)

```c
// Create execution context
MemoryContext exec_ctx = create_context("Executor", query_ctx);
switch_to_context(exec_ctx);

// Initialize executor state (tagged union for execution nodes)
SeqScanState *scan_state = makeNode(SeqScanState);
scan_state->type = T_SeqScanState;
scan_state->plan = plan;

// Create per-tuple context (child of exec_ctx)
MemoryContext tuple_ctx = create_context("Per-Tuple", exec_ctx);

// Execute scan using method tables (table access methods)
TableAmRoutine *heap_methods = get_table_methods(relation);

while (more_rows) {
    // Switch to tuple context
    reset_context(tuple_ctx);  // Free previous tuple's temp allocations
    switch_to_context(tuple_ctx);

    // Fetch tuple using method table
    TupleTableSlot *slot = heap_methods->scan_getnextslot(scan_state);

    // Process tuple...
    // All temp allocations go in tuple_ctx

    // tuple_ctx will be reset at next iteration
}

// When query finishes:
delete_context(query_ctx);
// This automatically deletes:
//   - parse_ctx (and all parse tree nodes)
//   - plan_ctx (and all plan nodes)
//   - exec_ctx (and all execution state)
//   - tuple_ctx (and all per-tuple allocations)
```

**What we just saw:**
1. **Tagged unions** - SelectStmt → SeqScan → SeqScanState (all use same base Node type)
2. **Memory contexts** - Hierarchical cleanup (query_ctx → parse/plan/exec → tuple)
3. **Method tables** - Index cost estimation, table scanning (pluggable implementations)
4. **Code generation** - All the makeNode/copyObject functions work automatically

---

## Part 4: Why This Architecture Matters

### Problem 1: Adding a New SQL Statement

**Without these patterns:**
- Manually write parsing code
- Manually write copy function
- Manually write equality function
- Manually write serialization
- Update dozens of switch statements
- Track all allocations manually
- **Result:** Weeks of work, many bugs

**With these patterns:**
```c
// 1. Define the struct
typedef struct MyNewStmt {
    NodeTag type;
    char *my_field;
    int my_value;
} MyNewStmt;

// 2. Add to NodeTag enum
typedef enum {
    // ...
    T_MyNewStmt,
} NodeTag;

// 3. Run code generator - automatically creates:
//    - copy_MyNewStmt()
//    - equal_MyNewStmt()
//    - serialize_MyNewStmt()

// 4. Add parsing logic
case KEYWORD_MYNEW:
    MyNewStmt *stmt = makeNode(MyNewStmt);  // Allocated in current context
    stmt->my_field = parse_field();
    return stmt;

// 5. Add execution logic
case T_MyNewStmt:
    execute_my_new_stmt((MyNewStmt *)stmt);
    break;
```

**Result:** Hours of work instead of weeks, fewer bugs.

---

### Problem 2: Adding a New Index Type

**Without method tables:**
- Hard-coded switch statements everywhere
- Can't add types without modifying core code
- Tight coupling

**With method tables:**
```c
// 1. Implement the interface
IndexAmRoutine my_index_methods = {
    .ambuild = my_build_function,
    .aminsert = my_insert_function,
    .ambeginscan = my_scan_begin,
    .amgettuple = my_get_tuple,
    .amcostestimate = my_cost_function,
    // ... more methods
};

// 2. Register it
CREATE ACCESS METHOD my_index TYPE INDEX HANDLER my_index_handler;

// 3. Use it
CREATE INDEX idx ON table USING my_index (column);
```

Core PostgreSQL code doesn't change. Your index just works.

---

### Problem 3: Memory Leaks in Complex Operations

**Without hierarchical contexts:**
```c
void complex_function() {
    void *a = malloc(...);
    void *b = malloc(...);
    void *c = malloc(...);

    if (error1) { free(a); return; }
    if (error2) { free(a); free(b); return; }
    if (error3) { free(a); free(b); free(c); return; }

    // ... 50 more allocations ...
    // ... 50 more error checks ...
    // Tracking all this is impossible
}
```

**With hierarchical contexts:**
```c
void complex_function() {
    MemoryContext my_ctx = create_context("Work", CurrentContext);
    MemoryContext old_ctx = switch_to_context(my_ctx);

    void *a = palloc(...);  // Goes in my_ctx
    void *b = palloc(...);  // Goes in my_ctx
    void *c = palloc(...);  // Goes in my_ctx

    // ... 50 more allocations - all go in my_ctx

    // Any error? Context automatically deleted
    // No error? Delete it ourselves

    switch_to_context(old_ctx);
    delete_context(my_ctx);  // Everything freed, no leaks possible
}
```

---

## Part 5: The Big Picture

### How It All Fits Together

```
┌─────────────────────────────────────────────────┐
│  Query: SELECT * FROM users WHERE age > 21      │
└────────────┬────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────┐
│  PARSER                                        │
│  - Creates parse tree (SelectStmt nodes)      │
│  - Uses: Tagged unions, memory contexts       │
│  - Code generation: makeNode() works          │
└────────────┬───────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────┐
│  PLANNER                                       │
│  - Creates plan tree (SeqScan, IndexScan)     │
│  - Uses: Method tables (cost estimation)      │
│  - Uses: Tagged unions (different plan types) │
│  - Uses: Memory contexts (planning work)      │
└────────────┬───────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────┐
│  EXECUTOR                                      │
│  - Creates exec tree (SeqScanState, etc.)     │
│  - Uses: Method tables (table/index access)   │
│  - Uses: Hierarchical contexts (per-tuple)    │
│  - Uses: Function pointers (node dispatch)    │
└────────────┬───────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────┐
│  STORAGE                                       │
│  - Accesses heap tables, indexes              │
│  - Uses: Method tables (pluggable storage)    │
│  - Uses: Memory contexts (buffer management)  │
└────────────────────────────────────────────────┘
```

**At every stage:**
- **Tagged unions** represent the current tree
- **Memory contexts** manage allocations
- **Method tables** provide pluggable behavior
- **Code generation** handles repetitive operations

---

### Design Principles Summary

PostgreSQL follows these principles:

1. **Everything is a Node**
   - Uniform representation
   - Generic operations (copy, equal, serialize)
   - Type-safe casting with tags

2. **Everything has a Lifetime**
   - Grouped in memory contexts
   - Automatic cleanup
   - No manual tracking

3. **Everything is Pluggable**
   - Method tables for different implementations
   - No hard-coded behavior
   - Runtime selection

4. **Minimize Human Error**
   - Code generation for repetitive code
   - Hierarchical cleanup (can't forget)
   - Consistent patterns everywhere

5. **Separation of Concerns**
   - Parser only creates parse trees
   - Planner only creates plan trees
   - Executor only executes
   - Each stage has its own memory context

---

## Part 6: Reading the Code - A Practical Guide

Now that you understand the patterns, here's how to actually read PostgreSQL code:

### Starting Point 1: Following a Query

To understand query processing, trace this path:

**Entry point:** `src/backend/tcop/postgres.c:1000` - `exec_simple_query()`
```c
exec_simple_query(const char *query_string) {
    // 1. Parse
    List *parsetree_list = pg_parse_query(query_string);
    // Returns SelectStmt or other statement nodes

    // 2. Analyze and rewrite
    List *querytree_list = pg_analyze_and_rewrite(...);
    // Returns Query nodes

    // 3. Plan
    List *plantree_list = pg_plan_queries(...);
    // Returns PlannedStmt nodes

    // 4. Execute
    PortalRun(portal, ...);
}
```

**What to look for:**
- Memory context switches: `MemoryContextSwitchTo()`
- Node creation: `makeNode()`
- Type checks: `if (IsA(node, SelectStmt))`
- Method table calls: `->scan_getnextslot()`

### Starting Point 2: Understanding a Node Type

Pick any node type and examine:

1. **Definition** (e.g., `src/include/nodes/parsenodes.h`):
```c
typedef struct SelectStmt {
    NodeTag type;
    // Fields with attributes
} SelectStmt;
```

2. **Creation** (grep for `makeNode(SelectStmt)`):
```c
SelectStmt *stmt = makeNode(SelectStmt);
stmt->field = value;
```

3. **Processing** (grep for `T_SelectStmt` in switch statements):
```c
switch (nodeTag(node)) {
    case T_SelectStmt:
        process_select((SelectStmt *)node);
        break;
}
```

4. **Generated functions** (in `src/backend/nodes/`):
- `copyfuncs.c` - copy function
- `equalfuncs.c` - equality function
- `outfuncs.c` - serialization

### Starting Point 3: Understanding Memory Management

1. **Read:** `src/backend/utils/mmgr/README`
   - Explains context system
   - Describes different allocators
   - Shows usage patterns

2. **Examine context creation:**
```c
// Find: AllocSetContextCreate
MemoryContext ctx = AllocSetContextCreate(
    parent,
    "ContextName",
    ALLOCSET_DEFAULT_SIZES
);
```

3. **Examine context switching:**
```c
MemoryContext old = MemoryContextSwitchTo(new_ctx);
// ... allocations go in new_ctx ...
MemoryContextSwitchTo(old);
```

4. **Examine cleanup:**
```c
MemoryContextDelete(ctx);  // Deletes entire subtree
MemoryContextReset(ctx);   // Frees allocations, keeps context
```

### Starting Point 4: Understanding Method Tables

1. **Find the method table definition** (e.g., `src/include/access/tableam.h`):
```c
typedef struct TableAmRoutine {
    NodeTag type;

    // Scan methods
    TableScanDesc (*scan_begin)(...);
    bool (*scan_getnextslot)(...);
    void (*scan_end)(...);

    // Modify methods
    TM_Result (*tuple_insert)(...);
    TM_Result (*tuple_update)(...);
    TM_Result (*tuple_delete)(...);

    // ... more methods
} TableAmRoutine;
```

2. **Find an implementation** (e.g., `src/backend/access/heap/heapam_handler.c`):
```c
static const TableAmRoutine heapam_methods = {
    .type = T_TableAmRoutine,

    .scan_begin = heap_beginscan,
    .scan_getnextslot = heap_getnextslot,
    .scan_end = heap_endscan,

    .tuple_insert = heap_tuple_insert,
    .tuple_update = heap_tuple_update,
    .tuple_delete = heap_tuple_delete,

    // ... more implementations
};
```

3. **Find usage:**
```c
// Get the method table
TableAmRoutine *methods = relation->rd_tableam;

// Call through it
TupleTableSlot *slot = methods->scan_getnextslot(scan);
```

---

## Part 7: Common Patterns You'll See

### Pattern: Node Dispatch

You'll see this **everywhere**:
```c
switch (nodeTag(node)) {
    case T_TypeA:
        return process_type_a((TypeA *)node);
    case T_TypeB:
        return process_type_b((TypeB *)node);
    // ... dozens of cases
    default:
        elog(ERROR, "unrecognized node type: %d", nodeTag(node));
}
```

### Pattern: Memory Context Switching

```c
MemoryContext oldcontext = MemoryContextSwitchTo(mycontext);
// ... allocations ...
MemoryContextSwitchTo(oldcontext);
```

### Pattern: List Building

```c
List *list = NIL;  // Start with empty list
foreach(item, input_list) {
    Node *processed = process(lfirst(item));
    list = lappend(list, processed);
}
```

### Pattern: Function Table Calls

```c
// Get method table
SomeAmRoutine *routine = GetSomeAmRoutine(handler_oid);

// Call method
routine->some_method(args);
```

### Pattern: Tree Walking

```c
bool my_walker(Node *node, void *context) {
    if (node == NULL)
        return false;

    // Process this node
    if (interesting_node(node)) {
        do_something(node, context);
    }

    // Continue to children
    return expression_tree_walker(node, my_walker, context);
}
```

---

## Conclusion: Your Learning Path

### Beginner Level (You Are Here)
✅ Understand C basics (structs, pointers, malloc)
✅ Understand design patterns (tagged unions, method tables, hierarchical contexts)
✅ Understand how patterns combine in PostgreSQL

### Intermediate Level (Next Steps)
1. **Read a simple SQL path end-to-end**
   - Start: `exec_simple_query()`
   - Follow: parse → plan → execute
   - Focus on one statement type (e.g., SELECT)

2. **Pick one subsystem to understand deeply**
   - Option A: Parser (how SQL text becomes nodes)
   - Option B: Planner (how nodes become plans)
   - Option C: Executor (how plans execute)
   - Option D: Storage (how data is stored/retrieved)

3. **Read the READMEs**
   - Each subsystem has README files explaining design
   - `src/backend/optimizer/README`
   - `src/backend/executor/README`
   - `src/backend/utils/mmgr/README`

### Advanced Level (Future Goal)
1. Modify PostgreSQL to add features
2. Understand performance optimizations
3. Contribute to PostgreSQL development

---

## Quick Reference: Key Files

**Node system:**
- `src/include/nodes/nodes.h` - Base Node definition
- `src/include/nodes/parsenodes.h` - Parse tree nodes
- `src/include/nodes/plannodes.h` - Plan tree nodes
- `src/include/nodes/execnodes.h` - Execution tree nodes
- `src/backend/nodes/copyfuncs.c` - Generated copy functions
- `src/backend/nodes/equalfuncs.c` - Generated equality functions

**Memory management:**
- `src/backend/utils/mmgr/README` - Full explanation
- `src/backend/utils/mmgr/mcxt.c` - Context management
- `src/backend/utils/mmgr/aset.c` - AllocSet allocator
- `src/include/utils/memutils.h` - Memory context interface

**Query processing:**
- `src/backend/tcop/postgres.c` - Main query loop
- `src/backend/parser/parser.c` - Parsing entry point
- `src/backend/optimizer/plan/planner.c` - Planning entry point
- `src/backend/executor/execMain.c` - Execution entry point

**Access methods:**
- `src/include/access/tableam.h` - Table AM interface
- `src/include/access/amapi.h` - Index AM interface
- `src/backend/access/heap/heapam_handler.c` - Heap table implementation
- `src/backend/access/nbtree/nbtree.c` - B-tree index implementation

---

## Your First Exercise

To test your understanding, try this:

1. **Find the SelectStmt definition**
   - Location: `src/include/nodes/parsenodes.h`
   - Look for: `typedef struct SelectStmt`

2. **Find where it's created**
   - Search for: `makeNode(SelectStmt)`
   - Look in: `src/backend/parser/gram.y`

3. **Find where it's processed**
   - Search for: `T_SelectStmt` in switch statements
   - Look in: `src/backend/tcop/utility.c` or `src/backend/commands/`

4. **Find the generated functions**
   - Look in: `src/backend/nodes/copyfuncs.c` for `copy_SelectStmt`
   - Look in: `src/backend/nodes/equalfuncs.c` for `equal_SelectStmt`

This will give you a concrete path through the codebase using all the patterns we discussed.

---

**Remember:** PostgreSQL is huge. You don't need to understand everything. Pick one part, understand it deeply using these patterns, then expand to adjacent areas. The patterns are consistent throughout, so once you understand them in one place, you can apply them everywhere.
