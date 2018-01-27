---
layout: post
title: O(1)空间复杂度校验有序二叉树 
description:  leetcode二叉树题解
categories: leetcode tech
author: lambdae
---



*  **引言**

    这个很好解，中序遍历二叉树得到有序序列就是BST了，时间复杂度为O(n)即需要遍历整个树。优化点在于将空间复杂度降低到O(1), 同时，在中序遍历二叉树的时候，一旦有sub tree不满足BST的性质就结束遍历。只有在二叉树是BST或者最后的节点不满足BST的时候才会遍历整个树。


* **code**
    

     ```C
    int last, end, cnt;
    bool valid;

    void inorder(struct TreeNode *node) {
        if (!node)    return;
        inorder(node->left);
        end = node->val;
        if (++cnt > 1) {
            if (end <= last) {
                valid = false;
                return;
            }
        }
        last = end;
        inorder(node->right);
    }
    
    bool isValidBST(struct TreeNode *root) {
        cnt = 0;
        valid = true;
        inorder(root);
        
        return valid;
    }
    
     ```