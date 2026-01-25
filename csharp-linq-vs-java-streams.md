# C# LINQ vs Java Streams Quick Reference

## Table of Contents

- [Lambda Syntax](#lambda-syntax)
- [Method Comparison](#method-comparison)
- [Common Operations](#common-operations)
  - [Filter](#filter)
  - [Map / Transform](#map--transform)
  - [Sort](#sort)
  - [Find First](#find-first)
  - [Any / All](#any--all)
  - [Count](#count)
  - [Distinct](#distinct)
  - [Skip / Limit](#skip--limit)
  - [Group By](#group-by)
  - [Flat Map](#flat-map)
  - [Reduce / Aggregate](#reduce--aggregate)
  - [Collect to List](#collect-to-list)
  - [Collect to Map / Dictionary](#collect-to-map--dictionary)
- [LINQ Works on Any Collection](#linq-works-on-any-collection)
- [LINQ to Objects vs LINQ to Entities](#linq-to-objects-vs-linq-to-entities)
- [Query Syntax (SQL-like)](#query-syntax-sql-like)
- [Async LINQ (Entity Framework)](#async-linq-entity-framework)

---

## Lambda Syntax

| Java | C# |
|------|-----|
| `->` | `=>` |
| `x -> x.getName()` | `x => x.Name` |
| `(x, y) -> x + y` | `(x, y) => x + y` |
| `() -> doSomething()` | `() => DoSomething()` |

```java
// Java
list.forEach(x -> System.out.println(x));
```

```csharp
// C#
list.ForEach(x => Console.WriteLine(x));
```

---

## Method Comparison

| Java Streams | C# LINQ | Notes |
|--------------|---------|-------|
| `.stream()` | Not needed | LINQ is built into collections |
| `.filter()` | `.Where()` | |
| `.map()` | `.Select()` | |
| `.flatMap()` | `.SelectMany()` | |
| `.sorted()` | `.OrderBy()` | |
| `.sorted(Comparator.reverseOrder())` | `.OrderByDescending()` | |
| `.distinct()` | `.Distinct()` | |
| `.limit()` | `.Take()` | |
| `.skip()` | `.Skip()` | |
| `.count()` | `.Count()` | |
| `.findFirst()` | `.FirstOrDefault()` | Returns null if not found |
| `.findFirst().orElseThrow()` | `.First()` | Throws if not found |
| `.findAny()` | `.FirstOrDefault()` | |
| `.anyMatch()` | `.Any()` | |
| `.allMatch()` | `.All()` | |
| `.noneMatch()` | `!.Any()` or `.All(x => !condition)` | |
| `.max()` | `.Max()` | |
| `.min()` | `.Min()` | |
| `.reduce()` | `.Aggregate()` | |
| `.collect(Collectors.toList())` | `.ToList()` | |
| `.collect(Collectors.toSet())` | `.ToHashSet()` | |
| `.collect(Collectors.toMap())` | `.ToDictionary()` | |
| `.collect(Collectors.groupingBy())` | `.GroupBy()` | |
| `.forEach()` | `.ForEach()` (List only) or `foreach` loop | |
| `.peek()` | No direct equivalent | Use `Select` with side effect |
| `.parallel()` | `.AsParallel()` | PLINQ |

---

## Common Operations

### Filter

```java
// Java
List<User> adults = users.stream()
    .filter(u -> u.getAge() > 18)
    .collect(Collectors.toList());
```

```csharp
// C#
var adults = users
    .Where(u => u.Age > 18)
    .ToList();
```

---

### Map / Transform

```java
// Java
List<String> names = users.stream()
    .map(u -> u.getName())
    .collect(Collectors.toList());

// Method reference
List<String> names = users.stream()
    .map(User::getName)
    .collect(Collectors.toList());
```

```csharp
// C#
var names = users
    .Select(u => u.Name)
    .ToList();
```

---

### Sort

```java
// Java - Ascending
List<User> sorted = users.stream()
    .sorted(Comparator.comparing(User::getName))
    .collect(Collectors.toList());

// Java - Descending
List<User> sorted = users.stream()
    .sorted(Comparator.comparing(User::getName).reversed())
    .collect(Collectors.toList());

// Java - Multiple fields
List<User> sorted = users.stream()
    .sorted(Comparator.comparing(User::getLastName)
                      .thenComparing(User::getFirstName))
    .collect(Collectors.toList());
```

```csharp
// C# - Ascending
var sorted = users
    .OrderBy(u => u.Name)
    .ToList();

// C# - Descending
var sorted = users
    .OrderByDescending(u => u.Name)
    .ToList();

// C# - Multiple fields
var sorted = users
    .OrderBy(u => u.LastName)
    .ThenBy(u => u.FirstName)
    .ToList();
```

---

### Find First

```java
// Java
Optional<User> user = users.stream()
    .filter(u -> u.getId() == 1)
    .findFirst();

// With default
User user = users.stream()
    .filter(u -> u.getId() == 1)
    .findFirst()
    .orElse(null);

// Throw if not found
User user = users.stream()
    .filter(u -> u.getId() == 1)
    .findFirst()
    .orElseThrow(() -> new NotFoundException("User not found"));
```

```csharp
// C# - Returns null if not found
var user = users.FirstOrDefault(u => u.Id == 1);

// C# - Throws if not found
var user = users.First(u => u.Id == 1);

// C# - With separate filter
var user = users
    .Where(u => u.Id == 1)
    .FirstOrDefault();
```

---

### Any / All

```java
// Java
boolean hasAdmin = users.stream().anyMatch(u -> u.isAdmin());
boolean allActive = users.stream().allMatch(u -> u.isActive());
boolean noneExpired = users.stream().noneMatch(u -> u.isExpired());
```

```csharp
// C#
bool hasAdmin = users.Any(u => u.IsAdmin);
bool allActive = users.All(u => u.IsActive);
bool noneExpired = !users.Any(u => u.IsExpired);
// or
bool noneExpired = users.All(u => !u.IsExpired);
```

---

### Count

```java
// Java
long count = users.stream()
    .filter(u -> u.isActive())
    .count();
```

```csharp
// C#
int count = users.Count(u => u.IsActive);

// or
int count = users
    .Where(u => u.IsActive)
    .Count();
```

---

### Distinct

```java
// Java
List<String> uniqueNames = users.stream()
    .map(User::getName)
    .distinct()
    .collect(Collectors.toList());
```

```csharp
// C#
var uniqueNames = users
    .Select(u => u.Name)
    .Distinct()
    .ToList();
```

---

### Skip / Limit

```java
// Java
List<User> page = users.stream()
    .skip(10)
    .limit(5)
    .collect(Collectors.toList());
```

```csharp
// C#
var page = users
    .Skip(10)
    .Take(5)
    .ToList();
```

---

### Group By

```java
// Java
Map<String, List<User>> byDepartment = users.stream()
    .collect(Collectors.groupingBy(User::getDepartment));

// With downstream collector
Map<String, Long> countByDepartment = users.stream()
    .collect(Collectors.groupingBy(User::getDepartment, Collectors.counting()));
```

```csharp
// C#
var byDepartment = users
    .GroupBy(u => u.Department)
    .ToDictionary(g => g.Key, g => g.ToList());

// Just grouping (IGrouping)
var grouped = users.GroupBy(u => u.Department);
foreach (var group in grouped)
{
    Console.WriteLine($"{group.Key}: {group.Count()}");
}

// Count by department
var countByDepartment = users
    .GroupBy(u => u.Department)
    .ToDictionary(g => g.Key, g => g.Count());
```

---

### Flat Map

**What is flattening?**
When you have a list of objects, and each object contains another list,
`flatMap`/`SelectMany` merges all inner lists into one single list.

```
Before (nested):     [ [A, B], [C], [D, E, F] ]
After (flattened):   [ A, B, C, D, E, F ]
```

**Example types:**
```csharp
// A Post has a list of Tags
public class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public List<string> Tags { get; set; }  // Each post has multiple tags
}

// Sample data
var posts = new List<Post>
{
    new Post { Id = 1, Title = "C# Tips",   Tags = new List<string> { "csharp", "dotnet" } },
    new Post { Id = 2, Title = "API Guide", Tags = new List<string> { "api", "rest", "dotnet" } },
    new Post { Id = 3, Title = "Docker 101", Tags = new List<string> { "docker", "devops" } }
};

// Without flattening - you get List<List<string>>
// posts.Select(p => p.Tags) → [ ["csharp","dotnet"], ["api","rest","dotnet"], ["docker","devops"] ]

// With flattening - you get List<string>
// posts.SelectMany(p => p.Tags) → [ "csharp", "dotnet", "api", "rest", "dotnet", "docker", "devops" ]
```

```java
// Java
List<String> allTags = posts.stream()
    .flatMap(post -> post.getTags().stream())  // Flatten each post's tags into one stream
    .distinct()                                  // Remove duplicates
    .collect(Collectors.toList());
// Result: ["csharp", "dotnet", "api", "rest", "docker", "devops"]
```

```csharp
// C#
var allTags = posts
    .SelectMany(post => post.Tags)  // Flatten each post's tags into one list
    .Distinct()                      // Remove duplicates
    .ToList();
// Result: ["csharp", "dotnet", "api", "rest", "docker", "devops"]
```

**Visual breakdown:**
```
Post 1: Tags = ["csharp", "dotnet"]
Post 2: Tags = ["api", "rest", "dotnet"]
Post 3: Tags = ["docker", "devops"]

Select (no flatten):     [ ["csharp","dotnet"], ["api","rest","dotnet"], ["docker","devops"] ]
SelectMany (flattened):  [ "csharp", "dotnet", "api", "rest", "dotnet", "docker", "devops" ]
Distinct:                [ "csharp", "dotnet", "api", "rest", "docker", "devops" ]
```

---

### Reduce / Aggregate

```java
// Java
int sum = numbers.stream()
    .reduce(0, (a, b) -> a + b);

// With method reference
int sum = numbers.stream()
    .reduce(0, Integer::sum);

// String concatenation
String combined = strings.stream()
    .reduce("", (a, b) -> a + b);
```

```csharp
// C#
int sum = numbers.Aggregate(0, (a, b) => a + b);

// Simpler for sum
int sum = numbers.Sum();

// String concatenation
string combined = strings.Aggregate("", (a, b) => a + b);

// Or use string.Join
string combined = string.Join("", strings);
```

---

### Collect to List

```java
// Java
List<String> names = users.stream()
    .map(User::getName)
    .collect(Collectors.toList());

// Java 16+
List<String> names = users.stream()
    .map(User::getName)
    .toList();
```

```csharp
// C#
var names = users
    .Select(u => u.Name)
    .ToList();

// To array
var names = users
    .Select(u => u.Name)
    .ToArray();
```

---

### Collect to Map / Dictionary

```java
// Java
Map<Long, User> userById = users.stream()
    .collect(Collectors.toMap(User::getId, u -> u));

// With value mapping
Map<Long, String> nameById = users.stream()
    .collect(Collectors.toMap(User::getId, User::getName));
```

```csharp
// C#
var userById = users.ToDictionary(u => u.Id, u => u);

// With value mapping
var nameById = users.ToDictionary(u => u.Id, u => u.Name);

// Handle duplicates
var userById = users.ToDictionary(
    u => u.Id,
    u => u,
    (existing, replacement) => existing  // Keep first on duplicate
);
```

---

## LINQ Works on Any Collection

LINQ is not just for databases — it works on any `IEnumerable<T>`:

| Collection Type | Works with LINQ |
|-----------------|-----------------|
| `List<T>` | Yes |
| `T[]` (Array) | Yes |
| `Dictionary<K,V>` | Yes |
| `HashSet<T>` | Yes |
| `Queue<T>` | Yes |
| `DbSet<T>` (EF) | Yes |
| Any `IEnumerable<T>` | Yes |

```csharp
// On List
var result = myList.Where(x => x.Active).ToList();

// On Array
int[] numbers = { 1, 2, 3, 4, 5 };
var evens = numbers.Where(n => n % 2 == 0).ToArray();

// On Dictionary
var filtered = myDict.Where(kv => kv.Value > 10);
```

---

## LINQ to Objects vs LINQ to Entities

| Type | Execution | Use Case |
|------|-----------|----------|
| **LINQ to Objects** | In-memory | Lists, arrays, any collection |
| **LINQ to Entities** | Translated to SQL | Entity Framework database queries |

```csharp
// LINQ to Objects (in-memory)
var result = myList.Where(x => x.Active).ToList();

// LINQ to Entities (becomes SQL)
var result = _dbContext.Users.Where(x => x.Active).ToList();
// SQL: SELECT * FROM Users WHERE Active = 1
```

**Same syntax, different execution** — this is the power of LINQ.

---

## Query Syntax (SQL-like)

C# has an alternative SQL-like syntax (same result as method syntax):

```csharp
// Method syntax (recommended, like Java Streams)
var result = users
    .Where(u => u.Age > 18)
    .OrderBy(u => u.Name)
    .Select(u => u.Name);

// Query syntax (SQL-like)
var result = from u in users
             where u.Age > 18
             orderby u.Name
             select u.Name;

// Query syntax with join
var result = from u in users
             join d in departments on u.DeptId equals d.Id
             where u.Active
             select new { u.Name, d.DepartmentName };
```

Most developers prefer **method syntax** — it's closer to Java Streams.

---

## Async LINQ (Entity Framework)

For database queries, use async versions:

```csharp
// Sync (blocks thread)
var users = _dbContext.Users
    .Where(u => u.Active)
    .ToList();

// Async (releases thread)
var users = await _dbContext.Users
    .Where(u => u.Active)
    .ToListAsync();

// Other async methods
var user = await _dbContext.Users.FirstOrDefaultAsync(u => u.Id == id);
var count = await _dbContext.Users.CountAsync(u => u.Active);
var exists = await _dbContext.Users.AnyAsync(u => u.Email == email);
```

**Note:** Async LINQ methods (`ToListAsync`, `FirstOrDefaultAsync`, etc.) are from Entity Framework, not available for in-memory collections.
