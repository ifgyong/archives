1.最深二叉树

```
//递归遍历
递归遍历简单点说就是上边的的深度等于下边的深度+1，关键代码是 1 + maxDepth(r);
func maxDepth(_ root: TreeNode?) -> Int {
        var left = 1;
        var right = 1;
        if root == nil {//为空的时候深度为0
            return 0;
        }
        if let r = root?.left {
            left = 1 + maxDepth(r);
            if let r = root?.right {
                right = 1 + maxDepth(r);
            }
            return left > right ? left : right;
        }else if let r = root?.right {
            right = 1 + maxDepth(r);
            return left > right ? left : right;
        }else {
          //没有子树的时候深度为1
            return 1;
        }
    }
```
