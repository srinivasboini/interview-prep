# Sorting Algorithms - Complete Master Guide

## Overview
While production code rarely implements sorting from scratch (use `Arrays.sort()` or `Collections.sort()`), understanding sorting algorithms is crucial for:
- Choosing the right algorithm for specific constraints
- Understanding time/space complexity trade-offs
- Recognizing when stability matters
- Optimizing performance-critical code

**Key Insight**: Different sorting algorithms excel in different scenarios—know when to use each.

For Senior/Staff Engineers, mastering sorting means:
- Understanding all major sorting algorithms
- Knowing Java's sorting internals
- Recognizing when to use stable vs unstable sorts
- Discussing production applications (database indexing, log processing)

---

## Table of Contents
1. [Comparison of Algorithms](#comparison-of-algorithms)
2. [Sorting Algorithms](#sorting-algorithms)
3. [Java Internals](#java-internals)
4. [10+ Solved Problems](#solved-problems)
5. [Interview Questions & Answers](#interview-questions--answers)
6. [Banking & Production Context](#banking--production-context)

---

## Comparison of Algorithms

| Algorithm | Time (Best) | Time (Avg) | Time (Worst) | Space | Stable? | Notes |
|-----------|-------------|------------|--------------|-------|---------|-------|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | Yes | Simple, rarely used |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | No | Minimal swaps |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | Yes | Good for small/nearly sorted |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes | Guaranteed O(n log n) |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | No | Fast in practice |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | No | In-place, guaranteed |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | Yes | Integer sorting |
| Radix Sort | O(d(n+k)) | O(d(n+k)) | O(d(n+k)) | O(n+k) | Yes | Integer sorting |
| Tim Sort | O(n) | O(n log n) | O(n log n) | O(n) | Yes | Python/Java default |

---

## Sorting Algorithms

### Bubble Sort

```java
/**
 * Bubble Sort - repeatedly swap adjacent elements.
 * Time: O(n²), Space: O(1), Stable: Yes
 */
public void bubbleSort(int[] arr) {
    int n = arr.length;
    
    for (int i = 0; i < n - 1; i++) {
        boolean swapped = false;
        
        for (int j = 0; j < n - i - 1; j++) {
            if (arr[j] > arr[j + 1]) {
                swap(arr, j, j + 1);
                swapped = true;
            }
        }
        
        if (!swapped) break;  // Already sorted
    }
}
```

### Selection Sort

```java
/**
 * Selection Sort - find minimum and place at beginning.
 * Time: O(n²), Space: O(1), Stable: No
 */
public void selectionSort(int[] arr) {
    int n = arr.length;
    
    for (int i = 0; i < n - 1; i++) {
        int minIdx = i;
        
        for (int j = i + 1; j < n; j++) {
            if (arr[j] < arr[minIdx]) {
                minIdx = j;
            }
        }
        
        swap(arr, i, minIdx);
    }
}
```

### Insertion Sort

```java
/**
 * Insertion Sort - build sorted array one element at a time.
 * Time: O(n²) worst, O(n) best, Space: O(1), Stable: Yes
 */
public void insertionSort(int[] arr) {
    int n = arr.length;
    
    for (int i = 1; i < n; i++) {
        int key = arr[i];
        int j = i - 1;
        
        while (j >= 0 && arr[j] > key) {
            arr[j + 1] = arr[j];
            j--;
        }
        
        arr[j + 1] = key;
    }
}
```

### Merge Sort

```java
/**
 * Merge Sort - divide and conquer.
 * Time: O(n log n), Space: O(n), Stable: Yes
 */
public void mergeSort(int[] arr) {
    if (arr.length <= 1) return;
    mergeSort(arr, 0, arr.length - 1);
}

private void mergeSort(int[] arr, int left, int right) {
    if (left >= right) return;
    
    int mid = left + (right - left) / 2;
    mergeSort(arr, left, mid);
    mergeSort(arr, mid + 1, right);
    merge(arr, left, mid, right);
}

private void merge(int[] arr, int left, int mid, int right) {
    int[] temp = new int[right - left + 1];
    int i = left, j = mid + 1, k = 0;
    
    while (i <= mid && j <= right) {
        if (arr[i] <= arr[j]) {
            temp[k++] = arr[i++];
        } else {
            temp[k++] = arr[j++];
        }
    }
    
    while (i <= mid) temp[k++] = arr[i++];
    while (j <= right) temp[k++] = arr[j++];
    
    System.arraycopy(temp, 0, arr, left, temp.length);
}
```

### Quick Sort

```java
/**
 * Quick Sort - partition around pivot.
 * Time: O(n log n) avg, O(n²) worst, Space: O(log n), Stable: No
 */
public void quickSort(int[] arr) {
    quickSort(arr, 0, arr.length - 1);
}

private void quickSort(int[] arr, int left, int right) {
    if (left >= right) return;
    
    int pivotIndex = partition(arr, left, right);
    quickSort(arr, left, pivotIndex - 1);
    quickSort(arr, pivotIndex + 1, right);
}

private int partition(int[] arr, int left, int right) {
    int pivot = arr[right];
    int i = left - 1;
    
    for (int j = left; j < right; j++) {
        if (arr[j] <= pivot) {
            i++;
            swap(arr, i, j);
        }
    }
    
    swap(arr, i + 1, right);
    return i + 1;
}
```

### Heap Sort

```java
/**
 * Heap Sort - build max heap and extract max.
 * Time: O(n log n), Space: O(1), Stable: No
 */
public void heapSort(int[] arr) {
    int n = arr.length;
    
    // Build max heap
    for (int i = n / 2 - 1; i >= 0; i--) {
        heapify(arr, n, i);
    }
    
    // Extract elements from heap
    for (int i = n - 1; i > 0; i--) {
        swap(arr, 0, i);
        heapify(arr, i, 0);
    }
}

private void heapify(int[] arr, int n, int i) {
    int largest = i;
    int left = 2 * i + 1;
    int right = 2 * i + 2;
    
    if (left < n && arr[left] > arr[largest]) {
        largest = left;
    }
    
    if (right < n && arr[right] > arr[largest]) {
        largest = right;
    }
    
    if (largest != i) {
        swap(arr, i, largest);
        heapify(arr, n, largest);
    }
}
```

### Counting Sort

```java
/**
 * Counting Sort - count occurrences of each value.
 * Time: O(n+k), Space: O(k), Stable: Yes
 * Works when range k is small.
 */
public void countingSort(int[] arr) {
    if (arr.length == 0) return;
    
    int max = Arrays.stream(arr).max().getAsInt();
    int min = Arrays.stream(arr).min().getAsInt();
    int range = max - min + 1;
    
    int[] count = new int[range];
    int[] output = new int[arr.length];
    
    // Count occurrences
    for (int num : arr) {
        count[num - min]++;
    }
    
    // Cumulative count
    for (int i = 1; i < range; i++) {
        count[i] += count[i - 1];
    }
    
    // Build output array
    for (int i = arr.length - 1; i >= 0; i--) {
        output[count[arr[i] - min] - 1] = arr[i];
        count[arr[i] - min]--;
    }
    
    System.arraycopy(output, 0, arr, 0, arr.length);
}
```

---

## Java Internals

### Arrays.sort()

**Primitives** (int[], double[], etc.):
- **Algorithm**: Dual-Pivot Quicksort
- **Time**: O(n log n) average
- **Stable**: No
- **Why**: Faster than merge sort, in-place

**Objects** (Integer[], String[], etc.):
- **Algorithm**: TimSort (merge sort + insertion sort hybrid)
- **Time**: O(n log n)
- **Stable**: Yes
- **Why**: Stability matters for objects

### Collections.sort()

- **Algorithm**: TimSort
- **Stable**: Yes
- **Converts to array**: Internally uses `Arrays.sort()`

---

## Solved Problems

### Problem 1: Sort Colors (Medium)

```java
/**
 * Sort array with values 0, 1, 2 (Dutch National Flag).
 * Time: O(n), Space: O(1)
 */
public void sortColors(int[] nums) {
    int low = 0, mid = 0, high = nums.length - 1;
    
    while (mid <= high) {
        if (nums[mid] == 0) {
            swap(nums, low++, mid++);
        } else if (nums[mid] == 1) {
            mid++;
        } else {
            swap(nums, mid, high--);
        }
    }
}
```

### Problem 2: Merge Sorted Array (Easy)

```java
/**
 * Merge nums2 into nums1.
 * Time: O(m+n), Space: O(1)
 */
public void merge(int[] nums1, int m, int[] nums2, int n) {
    int i = m - 1, j = n - 1, k = m + n - 1;
    
    while (i >= 0 && j >= 0) {
        if (nums1[i] > nums2[j]) {
            nums1[k--] = nums1[i--];
        } else {
            nums1[k--] = nums2[j--];
        }
    }
    
    while (j >= 0) {
        nums1[k--] = nums2[j--];
    }
}
```

### Problem 3: K Closest Points to Origin (Medium)

```java
/**
 * Find K closest points using sorting.
 * Time: O(n log n), Space: O(1)
 */
public int[][] kClosest(int[][] points, int k) {
    Arrays.sort(points, (a, b) -> {
        int dist1 = a[0] * a[0] + a[1] * a[1];
        int dist2 = b[0] * b[0] + b[1] * b[1];
        return dist1 - dist2;
    });
    
    return Arrays.copyOfRange(points, 0, k);
}
```

### Problem 4: Largest Number (Medium)

```java
/**
 * Arrange numbers to form largest number.
 * Time: O(n log n), Space: O(n)
 */
public String largestNumber(int[] nums) {
    String[] strs = new String[nums.length];
    for (int i = 0; i < nums.length; i++) {
        strs[i] = String.valueOf(nums[i]);
    }
    
    Arrays.sort(strs, (a, b) -> (b + a).compareTo(a + b));
    
    if (strs[0].equals("0")) return "0";
    
    return String.join("", strs);
}
```

---

## Interview Questions & Answers

### Q1: "When should you use merge sort vs quick sort?"

**Model Answer:**
"I choose based on requirements:

**Merge Sort**:
- **Pros**: Guaranteed O(n log n), stable, predictable
- **Cons**: O(n) space, slower in practice
- **Use when**:
  - Stability required
  - Guaranteed performance needed
  - Linked lists (no random access penalty)
  - External sorting (large data on disk)

**Quick Sort**:
- **Pros**: O(1) space, faster in practice, cache-friendly
- **Cons**: O(n²) worst case, unstable
- **Use when**:
  - Memory constrained
  - Average case matters
  - Arrays (random access available)

**Production example**:
In banking:
- Use merge sort for transaction logs (stability for timestamp ties)
- Use quick sort for in-memory price sorting (memory efficient)

**Java's choice**:
- Primitives: Dual-Pivot Quicksort (speed)
- Objects: TimSort (stability)"

### Q2: "What is stability in sorting and why does it matter?"

**Model Answer:**
"Stability means equal elements maintain their relative order:

**Example**:
```
Input: [(Alice, 25), (Bob, 30), (Charlie, 25)]
Sort by age:

Stable: [(Alice, 25), (Charlie, 25), (Bob, 30)]
Unstable: [(Charlie, 25), (Alice, 25), (Bob, 30)]
```

**Why it matters**:

**1. Multi-key sorting**:
```java
// Sort by age, then by name
Collections.sort(people, Comparator.comparing(Person::getAge));
Collections.sort(people, Comparator.comparing(Person::getName));
// Stable sort preserves age order for same names
```

**2. Database ORDER BY**:
```sql
SELECT * FROM transactions 
ORDER BY amount, timestamp;
-- Stability ensures timestamp order for same amounts
```

**3. UI sorting**:
User sorts table by column A, then by column B—expects column A order preserved for equal B values.

**Stable algorithms**: Merge sort, insertion sort, bubble sort, counting sort  
**Unstable algorithms**: Quick sort, heap sort, selection sort

**Production example**:
In banking, displaying transactions:
- Sort by amount
- For same amount, preserve timestamp order
- Stability is crucial for audit trail"

### Q3: "Explain TimSort and why Java uses it."

**Model Answer:**
"TimSort is a hybrid sorting algorithm combining merge sort and insertion sort:

**Algorithm**:
1. Divide array into small runs (32-64 elements)
2. Sort each run with insertion sort
3. Merge runs using merge sort

**Why it's fast**:
- **Insertion sort**: O(n) for small/nearly sorted arrays
- **Merge sort**: O(n log n) guaranteed
- **Adaptive**: Exploits existing order in data

**Key optimizations**:
1. **Galloping mode**: When one run consistently wins during merge, use binary search
2. **Run detection**: Finds naturally ordered sequences
3. **Minimum run size**: Balances insertion sort vs merge sort

**Complexity**:
- **Best**: O(n) - already sorted
- **Average**: O(n log n)
- **Worst**: O(n log n)
- **Space**: O(n)
- **Stable**: Yes

**Why Java uses it**:
1. **Stability**: Required for object sorting
2. **Performance**: Faster than pure merge sort on real data
3. **Adaptive**: Exploits partial ordering
4. **Proven**: Used in Python, Java, Android

**Production example**:
In banking, sorting transaction logs:
- Often partially sorted (by timestamp)
- TimSort exploits this ordering
- Much faster than O(n log n) worst case"

---

## 🏦 Banking & Production Context

### Transaction Log Sorting

**Scenario**: Sort millions of transactions by multiple criteria.

```java
/**
 * Multi-criteria transaction sorting.
 */
class TransactionSorter {
    public void sortTransactions(List<Transaction> transactions) {
        // Multi-key sort: amount (desc), then timestamp (asc)
        Collections.sort(transactions, 
            Comparator.comparing(Transaction::getAmount).reversed()
                     .thenComparing(Transaction::getTimestamp));
    }
    
    // Custom stable sort for specific requirements
    public void customSort(List<Transaction> transactions) {
        // Use merge sort for stability
        mergeSort(transactions, 0, transactions.size() - 1);
    }
    
    private void mergeSort(List<Transaction> list, int left, int right) {
        if (left >= right) return;
        
        int mid = left + (right - left) / 2;
        mergeSort(list, left, mid);
        mergeSort(list, mid + 1, right);
        merge(list, left, mid, right);
    }
    
    private void merge(List<Transaction> list, int left, int mid, int right) {
        List<Transaction> temp = new ArrayList<>();
        int i = left, j = mid + 1;
        
        while (i <= mid && j <= right) {
            if (list.get(i).compareTo(list.get(j)) <= 0) {
                temp.add(list.get(i++));
            } else {
                temp.add(list.get(j++));
            }
        }
        
        while (i <= mid) temp.add(list.get(i++));
        while (j <= right) temp.add(list.get(j++));
        
        for (int k = 0; k < temp.size(); k++) {
            list.set(left + k, temp.get(k));
        }
    }
}
```

---

## Key Takeaways

1. **Comparison**: Know time/space/stability trade-offs
2. **Merge Sort**: O(n log n) guaranteed, stable, O(n) space
3. **Quick Sort**: O(n log n) average, unstable, O(1) space
4. **Heap Sort**: O(n log n) guaranteed, unstable, O(1) space
5. **Java**: Dual-Pivot Quicksort for primitives, TimSort for objects
6. **Stability**: Matters for multi-key sorting and audit trails
7. **Production**: Use `Arrays.sort()` or `Collections.sort()` unless specific constraints

---

**Next**: [Searching Algorithms](25-searching-algorithms.md)
