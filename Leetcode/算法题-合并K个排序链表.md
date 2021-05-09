## 算法题-合并K个排序链表

合并 k 个排序链表，返回合并后的排序链表。请分析和描述算法的复杂度。

示例:

输入:
[
  1->4->5,
  1->3->4,
  2->6
]
输出: 1->1->2->3->4->4->5->6

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/merge-k-sorted-lists
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

思路: 将整个链表组不断二分，直到只剩两个，合并他们，逐层上溯，得到最终结果。

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func mergeKLists(lists []*ListNode) *ListNode {
    // pay attention: dont always add it will make the arr too big for iterator 
    if len(lists)==0 {
        return nil
    }else if len(lists)==1 {
        return lists[0]
    }else if len(lists)==2 {
        return mergeTwoLists(lists[0],lists[1])
    }
    return mergeTwoLists(mergeKLists(lists[len(lists)/2:]),mergeKLists(lists[:len(lists)/2]))
}

func mergeTwoLists(l1 *ListNode, l2 *ListNode) *ListNode {
        var result ListNode
        temp := &result
        for {
            var node ListNode
            if l1==nil && l2==nil {
                break
            }else if (l1!=nil && l2!=nil&&l1.Val<l2.Val) || l2==nil {
                node.Val = l1.Val
                l1=l1.Next
            }else if (l1!=nil && l2!=nil&&l1.Val>=l2.Val) || l1==nil {
                node.Val = l2.Val
                l2=l2.Next
            }
            temp.Next=&node
            temp=temp.Next
        }
        return result.Next
}
```

