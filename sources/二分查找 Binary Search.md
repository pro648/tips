二分查找是最高效的算法之一，时间复杂度是`O(log n)`。与平衡的二叉搜索树复杂度一样。

想要使用二分查找，需满足以下条件：

- 集合必须能够在恒定时间查找任意索引的值。也就是集合需遵守`RandomAccessCollection`协议。
- 集合必须是有序的。

## 1. 示例

与线性查找相比，更能体现出二分查找的优势。Swift 数组的`firstIndex(of:)`方法使用线性查找，从index零开始遍历整个数组，直到查找到第一个匹配的元素。

![LinearSearch](images/21/BinarySearchLinearSearch.png)

Binary search 借助数据是有序的这一规则，以不同方式查找数据。下面是使用二分查找算法查找31的过程：

![BinarySearch](images/21/BinarySearch31.png)

线性查找需八步，而二分查找只需三步。二分查找步骤如下：

#### 1.1 找到中间元素

第一步，找到集合中间索引：

![MiddleIndex](images/21/BinarySearchMiddleIndex.png)

#### 1.2 中间元素与要查找值进行比较

中间元素与要查找值进行比较。如果匹配，直接返回该index；否则，进入下一步。

#### 1.3 递归调用二分查找

最后，递归调用二分查找，但只需查找中间元素左侧或右侧元素。如果要查找的元素小于中间元素，则查找中间元素左侧；反之，查找中间元素右侧。每次比较可排除一半元素。

例如查找31时，比中间元素22大，后续只查找右侧元素即可：

![RightSubsequence](images/21/BinarySearchRightSubsequence.png)

重复此过程，直到查找到元素，或集合不能再次被分割。

二分查找时间复杂度为`O(log n)`。

## 2. 实现二分查找 Binary Search

创建 playground，添加`BinarySearch.swift`文件，并添加以下代码：

```
public extension RandomAccessCollection where Element: Comparable {
    func binarySearch(for value: Element, in range: Range<Index>? = nil) -> Index? {
        // TODO:
    }
}
```

由于二分查找只能用于遵守`RandomAccessCollection`的类型，将二分查找方法添加到`RandomAccessCollection`的extension，同时限制元素必须可比较。

查找过程中会递归调用该方法，因此传入`range`参数。其为可选类型，方便搜索整个集合时不传入`range`。

下面实现搜索逻辑：

```
    func binarySearch(for value: Element, in range: Range<Index>? = nil) -> Index? {
        // 如果range为nil，搜索整个集合。
        let range = range ?? startIndex..<endIndex
        
        // 集合至少包含一个元素。
        guard range.lowerBound < range.upperBound else {
            return nil
        }
        
        // 查找中间索引
        let size = distance(from: range.lowerBound, to: range.upperBound)
        let middle = index(range.lowerBound, offsetBy: size / 2)
        
        if self[middle] == value {  // 如果与中间索引对应值匹配，直接返回。
            return middle
        } else if self[middle] > value {    // 如果没有匹配，继续查找左侧或右侧。
            return binarySearch(for: value, in: range.lowerBound..<middle)
        } else {
            return binarySearch(for: value, in: index(after: middle)..<range.upperBound)
        }
    }
```

使用以下代码进行测试：

```
let array = [1, 4, 6, 8, 11, 12, 18, 64, 80, 101]
let search11 = array.firstIndex(of: 11)
let binarySearch11 = array.binarySearch(for: 11)

print("firstIndex(of:): \(String(describing: search11))")
print("binarySearch(for:): \(String(describing: binarySearch11))")
```

输出如下：

```
firstIndex(of:): Optional(4)
binarySearch(for:): Optional(4)
```

二分查找非常高效，遇到有序数组查找元素时，应优先考虑二分查找。非有序数组只能线性查找，复杂度是`O(n²)`，可以考虑先排序，再使用二分查找的方式降低复杂度。

## 3. 二分查找算法题

#### 3.1 将二分查找算法作为独立函数

之前，将 binary search 算法作为`RandomAccessCollection`的extension使用，但由于二分查找只适用于有序数组，将该算法暴露到`RandomAccessCollection`协议可能导致误用。如何将其作为独立函数？

算法如下：

```
func binarySearch<Elements: RandomAccessCollection>(for element: Elements.Element, in collection: Elements, in range: Range<Elements.Index>? = nil) -> Elements.Index? where Elements.Element: Comparable {
    let range = range ?? collection.startIndex..<collection.endIndex
    guard range.lowerBound < range.upperBound else {
        return nil
    }
    let size = collection.distance(from: range.lowerBound, to: range.upperBound)
    let middle = collection.index(range.lowerBound, offsetBy: size / 2)
    if collection[middle] == element {
        return middle
    } else if collection[middle] > element {
        return binarySearch(for: element, in: collection, in: range.lowerBound..<middle)
    } else {
        return binarySearch(for: element, in: collection, in: collection.index(after: middle)..<range.upperBound)
    }
}
```

#### 3.2 查找范围

查找有序数组中指定元素的范围，例如：

```
let array = [1, 2, 3, 3, 3, 4, 5, 5]
if let indices = findIndices(of: 3, in: array) {
    print(indices)
}
```

可以使用数组的`firstIndex(of:)`、`lastIndex(of:)`查找首个、最后一个元素索引，但其时间复杂度是`O(n)`。

遇到有序数组应考虑二分查找。前面实现的查找只能找到指定元素，但不能定位元素在其区间的位置。更新查找算法如下：

```
func findIndices(of value: Int, in array: [Int]) -> CountableRange<Int>? {
    guard let startIndex = startIndex(of: value, in: array, range: 0..<array.count) else { return nil }
    
    guard let endIndex = endIndex(of: value, in: array, range: 0..<array.count) else { return nil }
    
    return startIndex..<endIndex
}

func startIndex(of value: Int, in array: [Int], range: CountableRange<Int>) -> Int? {
    let middleIndex = range.lowerBound + (range.upperBound - range.lowerBound) / 2
    
    // 如果middleIndex是0或最后一个索引，无需继续递归。
    if middleIndex == 0 || middleIndex == array.count - 1 {
        if array[middleIndex] == value {
            return middleIndex
        } else {
            return nil
        }
    }
    
    if array[middleIndex] == value {    // 如果middleIndex值与指定值相等
        if array[middleIndex - 1] != value {    // 如果中间索引前一个不相等，直接返回。
            return middleIndex
        } else {
            return startIndex(of: value, in: array, range: range.lowerBound..<middleIndex)
        }
    } else if value < array[middleIndex] {
        return startIndex(of: value, in: array, range: range.lowerBound..<middleIndex)
    } else {
        return startIndex(of: value, in: array, range: middleIndex..<range.upperBound)
    }
}

func endIndex(of value: Int, in array: [Int], range: CountableRange<Int>) -> Int? {
    let middleIndex = range.lowerBound + (range.upperBound - range.lowerBound) / 2
    
    if middleIndex == 0 || middleIndex == array.count - 1 {
        if array[middleIndex] == value {
            return middleIndex + 1
        } else {
            return nil
        }
    }
    
    if array[middleIndex] == value {
        if array[middleIndex + 1] != value {
            return middleIndex + 1
        } else {
            return endIndex(of: value, in: array, range: middleIndex..<range.upperBound)
        }
    } else if value < array[middleIndex] {
        return endIndex(of: value, in: array, range: range.lowerBound..<middleIndex)
    } else {
        return endIndex(of: value, in: array, range: middleIndex..<range.upperBound)
    }
}
```

使用以下代码进行测试：

```
let array = [1, 2, 3, 3, 3, 4, 5, 5]
if let indices = findIndices(of: 3, in: array) {
    print(indices)
}
```

控制台输出如下：

```
2..<5
```

上述算法时间复杂度为`O(log n)`。

## 总结

二分查找只适用于有序集合，复杂度为`O(log n)`，线性查找复杂度为`O(n)`。因此，先将数组排序，再使用二分查找，性能可能也会优于线性查找。

Demo名称：BinarySearch  
源码地址：<https://github.com/pro648/BasicDemos-iOS/tree/master/BinarySearch>