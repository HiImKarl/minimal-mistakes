---
layout: single
title: "AVL Trees -- 1/4: Tree Definition and Insertion"
---

This is a beginner level guide for writing a basic implementation of an AVL tree in C++. All of the functionality of the tree will be contained in a single header file. I will assume that the reader is at least somewhat familiar with AVL trees.

You can download the full source code for the tutorial with git. I have tested the code on Linux, but macOS and Cygwin should be fine too. You will have to modify the makefile if you want to build it with MSVC. 

Open a terminal and run this in the directory you want to put the code and go to the directory for part1 (you will need to have [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) installed):
```
git clone git@github.com:HiImKarl/AVL-Trees.git
cd /part1
```

**Prerequisites**: 
1. Knowledge of C programming fundementals, i.e. pointers, explicit memory mangement.
2. Basic knowledge of C++ templates and synatx.
3. Understanding of the properties of an AVL tree. There are many resources online that you can look at. I will point you to some of them over the course of the tutorial.
4. I use a bit of big-O asymptotic worst-case notation throughout but honestly you can just skip those parts if you don't understand it.

I will implement the tree as a series of functions that will be passed the root node of the tree, mostly for code clarity.

**Potential Optimization**:  It would be better to create a tree class instead and hide logic so that you can provide a cleaner API.

# Node Definition

Let's try creating a tree with the following Node properties:

```cpp
template <typename T>
struct Node {
	typedef T value_type;

	Node(value_type value, Node<T> *left = nullptr, Node<T> *right = nullptr)
	: value(value), left(left), right(right), height(0) {}

	value_type value;
	Node<T> *left;
	Node<T> *right;
	size_t height;
};
```
For simplicity, we will be storing the height of tree instead of the balance factor. We define a leaf node's height to be 0.
 If memory is a concern, it is possible to store the BF using only two bits (though we would still be using a single byte). 

**Potential Optimization**: The size\_t data type is often implemented as unsigned long. Given that the number of nodes is roughly 2<sup>height</sup>, can you imagine how large a tree would need to be to require height in that range?

# Tree Height

Since we are already storing the height of each node, this is a trivial function. 
```cpp
template <typename T>
size_t Height(Node<T> *root) 
{
	return root->height;
}
 ```

# Finding Nodes

The tree is ordered, so walking through the tree and finding nodes is also quite simple. The height of the tree is O(log(n)), and in the worst case (finding a leaf node) ```Find``` will take time proportional to log(n). We will return a nullptr if the node cannot be found.

```cpp
template <typename T>
Node<T> *Find(Node<T> *root, typename Node<T>::value_type val) 
{
	Node<T> *walk = root;
	while (walk) {
		if (val == walk->value) break;
		if (val > walk->value) walk = walk->right;
		else walk = walk->left;
	}
	return walk;
}
```

# Tree Insertion

This function is a little bit more complicated. We want to take in a pointer to the root of the tree, as well as a value item to insert. Since the root of the tree may be rotated, it is necessary to return the (possibly different) root of the tree after the insertion. Since at most one rotation needs to be performed (in O(1)), this function takes time proportional to O(log(n)).

The first thing that we have to do is figure out where the new node should be placed. However, we will also need to keep track of each node we go through to find that place, because we may need to rotate them.

```cpp
template <typename T>
Node<T> *Insert(Node<T> *root, typename Node<T>::value_type val)
{
	// create a new node to insert into the tree,
	Node<T> *new_node = new Node<T>(val);
	// if root is null the new node will become the root node
	if (!root) return new_node;

	// find out where to put the new node
	// keep track of the route moved into the tree
	Node<T> **indirect = &root;
	Node<T> **parents[root->height + 1];
	size_t parents_index = 0;
	while (*indirect) {
		parents[parents_index++] = indirect;
		if ((*indirect)->value < val) indirect = &(*indirect)->right;
		else indirect = &(*indirect)->left;
	}

	// place the new node into tree
	*indirect = new_node;
	//...
```

We store pointers to the node pointers in the tree (which could be `node->left`, `node->right`, or `root`) since we will have to change their values in the case of rotations. 

Now, we have to traverse upwards from where we inserted the new node, update the height values, and perform rotations if necessary. We can bundle these two operations into one ```void check_for_rotations(Node<T> **node_ptr)``` method.

```cpp
	// update the height of the tree, applying rotations if necessary
	for (size_t i = 0; i < parents_index; ++i) 
		util::check_for_rotations(parents[parents_index - i - 1]);
	return root;
} // end of Insert()
```

**Potential Optimization**: I have left out a check to see if the height of a parent node is actually changed, because if isn't changed, further traversal up the tree is unecessary.

## Performing Rotations

We perform rotations if the balance factor of any node becomes -2 or 2, keeping in mind that the tree can only become unbalanced by 1 before we correct it. Here are nice [lecture slides](https://courses.cs.washington.edu/courses/cse373/06sp/handouts/lecture12.pdf) from Washington University to help you visualize what rotations are necessary. 

We will need to define a few more functions ```int get_balance_factor(Node<T> *node)```, ```void update_height(Node<T> **node_ptr```, ```void right_rotation(Node<T> **node_ptr)```, and the same for left, left-right, and right-left rotations. Notice that in our case, the balance factor is defined as left child height minus right child height.

We will deal with updating the height of the rotated nodes in the rotation functions, so it only has to be done here explicitly if the node is not rotated. 

```cpp
template <typename T>
void check_for_rotations(Node<T> **node_ptr)
{
	int bf = util::get_balance_factor(*node_ptr);
	// check BF and rotate appropriately
	if (bf >= 2) {
		int left_bf = get_balance_factor((*node_ptr)->right);
		if (left_bf > 0) left_rotation(node_ptr);
		else right_left_rotation(node_ptr);
	} else if (bf <= -2) {
		int right_bf = get_balance_factor((*node_ptr)->left); 
		if (right_bf < 0) right_rotation(node_ptr);
		else left_right_rotation(node_ptr);
	} else {
		update_height(*node_ptr);
	}
}
```

## Getting the Balance Factor
We defined the BF as the right child's height subtracted by the left.

```cpp
template <typename T>
int get_balance_factor(Node<T> *node) 
{
	long left_height = (node->left ? node->left->height + 1 : 0);
	long right_height = (node->right ? node->right->height + 1 : 0);
	return right_height - left_height;
}
```

## Updating Height
We take the larger of the heights between the left child and right child, and add 1 to get our current node's height. Notice how we take into account that the current node may be a leaf node.

```cpp
template <typename T>
void update_height(Node<T> *node) 
{
	size_t left_height = (node->left ? node->left->height + 1: 0);
	size_t right_height = (node->right ? node->right->height + 1: 0);
	node->height = (left_height > right_height ? left_height : right_height);
}
```

## Rotations

For the left rotation, the left child moves into the place of the head node, inheriting the head as its right child, and the head node inherits the left node's right subtree. Again, these [lecture slides](https://courses.cs.washington.edu/courses/cse373/06sp/handouts/lecture12.pdf) from Washington University are really helpful if you want to visualize this.

The right rotation is identical, except we are moving the right node into place of the head node, which inherits the right node's left subtree.
```cpp 
template <typename T>
void left_rotation(Node<T> **head) 
{
	Node<T> *rotated = *head;
	*head = (*head)->right;
	rotated->right = (*head)->left;
	(*head)->left = rotated;

	// update heights of the rotated nodes
	// the children of the subtrees are unmoved so
	// they do not need to be updated 
	update_height(rotated);
	update_height(*head);
}
```

The left-right rotations and right-left rotations constitute a rotation on the left or right (respectively) child nodes, then rotating the unbalanced node. The tree becomes unbalanced if the rotation is not done on the child node.

```cpp
template <typename T>
void right_left_rotation(Node<T> **head) 
{
	right_rotation(&(*head)->right);
	left_rotation(head);
}
```

This concludes the first part of the tutorial. Up [next](https://hiimkarl.github.io//Learning-AVL-Trees-2/), erasing nodes and verifying our tree works.
* [AVL 1](https://hiimkarl.github.io//Learning-AVL-Trees-1/)
* [AVL 2](https://hiimkarl.github.io//Learning-AVL-Trees-2/)
* [AVL 3](https://hiimkarl.github.io//Learning-AVL-Trees-3/)
* [AVL 4](https://hiimkarl.github.io//Learning-AVL-Trees-4/)
