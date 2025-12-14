# SQL JOIN Algorithm Principles:

This section focuses on the **JOIN Algorithm Principles**—how the database internally executes join operations—a crucial area for performance optimization and advanced interviewing.

We will use specific table data to simulate the steps involved in executing these three main algorithms.

## Table of Contents

- [Data Preparation](#data-preparation)
- [1. Nested Loop Join](#1-nested-loop-join)
- [2. Merge Join](#2-merge-join)
  - [Step 1: Sort Phase](#step-1)
  - [Step 2: Merge Phase](#step-2)
- [3. Hash Join](#3-hash-join)
  - [Step 1: Build Phase](#step-1-1)
  - [Step 2: Probe Phase](#step-2-1)

---

### Data Preparation {#data-preparation}

We are joining the **Customers** table and the **Orders** table to find the details of each customer's orders.

**Customers Table (The Smaller Table)**
This table has 4 rows and is assumed to be unsorted.


| Customer_ID | Name      |
| ------------- | ----------- |
| C103        | Xiao Ming |
| C101        | Xiao Hong |
| C104        | Xiao Gang |
| C102        | Xiao Li   |

**Orders Table (The Larger Table)**
This table has 6 rows and is assumed to be unsorted.


| Order_ID | Customer_ID (Foreign Key) | Product  |
| ---------- | --------------------------- | ---------- |
| O01      | C102                      | Computer |
| O02      | C104                      | Phone    |
| O03      | C101                      | Keyboard |
| O04      | C102                      | Mouse    |
| O05      | C104                      | Tablet   |
| O06      | C103                      | Printer  |

The SQL statement to be executed is:

```sql
SELECT T1.Name, T2.Product
FROM Customers T1
INNER JOIN Orders T2 
ON T1.Customer_ID = T2.Customer_ID;

```

---

### 1. Nested Loop Join {#1-nested-loop-join}

**Principle**: Take each row from the left table and **sequentially scan** the entire right table to find a match. This is the most straightforward, yet often the least efficient, method.


| Step               | Left Table Operation (Customers)    | Right Table Operation (Orders)                  | Result         | Scan Count                               |
| -------------------- | ------------------------------------- | ------------------------------------------------- | ---------------- | ------------------------------------------ |
| **1st Outer Loop** | Fetch first row: (C103, Xiao Ming)  | Scan Orders table, O01 to O06, looking for C103 | Found O06      | 6 times                                  |
| **2nd Outer Loop** | Fetch second row: (C101, Xiao Hong) | Scan Orders table, O01 to O06, looking for C101 | Found O03      | 6 times                                  |
| **3rd Outer Loop** | Fetch third row: (C104, Xiao Gang)  | Scan Orders table, O01 to O06, looking for C104 | Found O02, O05 | 6 times                                  |
| **4th Outer Loop** | Fetch fourth row: (C102, Xiao Li)   | Scan Orders table, O01 to O06, looking for C102 | Found O01, O04 | 6 times                                  |
| **Total Scans**    | 4 scans                             | 4 \times 6 = 24 scans                           | 6 result rows  | **30 Scans** (4 for Left + 24 for Right) |

**Summary**: If the left table has M rows and the right table has N rows, the total scan count is approximately M \times N times. **Note**: If the join key on the N table has an **index**, the inner loop can locate matches quickly, significantly improving efficiency.

### 2. Merge Join {#2-merge-join}

**Principle**: First, **sort** both tables based on the join key (Merge Phase), and then execute a simultaneous, parallel scan (Merge Phase), completing the match in a single pass.

#### Step 1: {#step-1}

Sort PhaseThe database must first sort both tables by `Customer_ID`.

**Customers Table - Sorted**


| Customer_ID | Name      |
| ------------- | ----------- |
| **C101**    | Xiao Hong |
| **C102**    | Xiao Li   |
| **C103**    | Xiao Ming |
| **C104**    | Xiao Gang |

**Orders Table - Sorted**


| Order_ID | Customer_ID (Foreign Key) | Product  |
| ---------- | --------------------------- | ---------- |
| O03      | **C101**                  | Keyboard |
| O01      | **C102**                  | Computer |
| O04      | **C102**                  | Mouse    |
| O06      | **C103**                  | Printer  |
| O02      | **C104**                  | Phone    |
| O05      | **C104**                  | Tablet   |

#### Step 2: {#step-2}

Merge PhaseThe database uses two pointers, starting from the beginning of both tables, moving **simultaneously downwards**.


| Customer Pointer | Order Pointer | Customer ID | Order ID | Result | Pointer Movement                         |
| ------------------ | --------------- | ------------- | ---------- | -------- | ------------------------------------------ |
| 1 (C101)         | 1 (C101)      | C101        | C101     | Match  | Both pointers move down                  |
| 2 (C102)         | 2 (C102)      | C102        | C102     | Match  | Order pointer moves down (to O04)        |
| 2 (C102)         | 3 (C102)      | C102        | C102     | Match  | Order pointer moves down (to O06)        |
| 3 (C103)         | 4 (C103)      | C103        | C103     | Match  | Both pointers move down                  |
| 4 (C104)         | 5 (C104)      | C104        | C104     | Match  | Order pointer moves down (to O05)        |
| 4 (C104)         | 6 (C104)      | C104        | C104     | Match  | Order pointer moves down (end of Orders) |

**Summary**: Sorting takes time, but once sorted, only M+N scans are needed for the merge. This method is generally faster than Nested Loop Join when both tables are large and **lack indexes/pre-sorting**.

### 3. Hash Join {#3-hash-join}

**Principle**: The smaller table (Build Table) is loaded into memory to create a **Hash Table**. The larger table (Probe Table) is then scanned, and its keys are used to probe the Hash Table in memory.

#### Step 1: {#step-1-1}

Build PhaseWe select the smaller **Customers** table to build a Hash Table in memory.


| Customer_ID | Storage Address (In-memory Bucket) | Stored Data       |
| ------------- | ------------------------------------ | ------------------- |
| C103        | HASH(C103)                         | (C103, Xiao Ming) |
| C101        | HASH(C101)                         | (C101, Xiao Hong) |
| C104        | HASH(C104)                         | (C104, Xiao Gang) |
| C102        | HASH(C102)                         | (C102, Xiao Li)   |

#### Step 2: {#step-2-1}

Probe PhaseScan the larger **Orders** table row by row, calculate the hash value of its `Customer_ID`, and look up the match in the in-memory Hash Table.


| Scanned Row (Orders) | Customer_ID | Hash Value | Lookup Result           | Matched Result       |
| ---------------------- | ------------- | ------------ | ------------------------- | ---------------------- |
| O01                  | C102        | HASH(C102) | Found (C102, Xiao Li)   | Xiao Li - Computer   |
| O02                  | C104        | HASH(C104) | Found (C104, Xiao Gang) | Xiao Gang - Phone    |
| O03                  | C101        | HASH(C101) | Found (C101, Xiao Hong) | Xiao Hong - Keyboard |
| O04                  | C102        | HASH(C102) | Found (C102, Xiao Li)   | Xiao Li - Mouse      |
| O05                  | C104        | HASH(C104) | Found (C104, Xiao Gang) | Xiao Gang - Tablet   |
| O06                  | C103        | HASH(C103) | Found (C103, Xiao Ming) | Xiao Ming - Printer  |

**Summary**: Hash Join is highly efficient for large datasets, especially when there's a significant size difference between the two tables. It avoids expensive sorting operations, but requires sufficient **memory** to hold the smaller hash table.
