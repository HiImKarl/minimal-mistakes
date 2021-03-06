---
layout: single
title: "AVL Trees -- 2/4: Erasing Nodes and Verification"
---

We will continue directly from where we last came off, completing the functionality for the tree.

# Erasing Nodes
Perhaps we would like to pass a node pointer as the argument to delete, but we would have to traverse the tree to find the node again anyways (as we may need to rotate these nodes), and that would just result in an extraneous call to ```Find()```. Instead, we will again pass an argument of ```value_type```.

Also, we cannot return ```NULL``` if a node with that value cannot be found, since if we delete the last node we must return a ```NULL``` for the empty tree. We will ignore this problem for now, but a possible solution would be passing an optional boolean reference that indicates whether the value was found (and deleted) or not.
 
Finding the node to delete is fairly simple; we need to keep an array of parent pointers similar to ```insert()```.
```cpp
template <typename T>
Node<T> *Erase(Node<T> *root, typename Node<T>::value_type val)
{
	if (!root) return root;
	Node<T> **parents[root->height + 1];
	size_t parents_index = 0;

	// find the node to delete
	Node<T> **indirect = &root;
	while (*indirect) {
		if (val == (*indirect)->value) break;
		parents[parents_index++] = indirect;
		if (val > (*indirect)->value) indirect = &(*indirect)->right;
		else indirect = &(*indirect)->left;
	}
	if (!(*indirect)) return root; 
	//...
```
If we are erasing a leaf node, it is intuitive to see that we would just need to traverse through its parents and correct their heights/apply rotations. 

However, what if we are deleting a node in the middle of the tree? Then we need to find a node that we _can_ remove (a leaf node) and we use it to replace the node we want to delete. 

To do so, we can either move to the left child (if it exists) and move right until there are no more right children, or move to the right and then move all the way to the left. This would find the largest node smaller than the node targeted for deletion, or the smallest node larger than the target node. We then perform the replacement and delete our replacement node.

```cpp
	//...
	// find a replacement 
	if ((*indirect)->left) {
		// need to check this node for imbalance as well
		parents[parents_index++] = indirect;
		Node<T> **walk = &(*indirect)->left;
		while ((*walk)->right) {
			parents[parents_index++] = walk;
			walk = &(*walk)->right;
		}
		// replace the value and then delete replacement node
		(*indirect)->value = (*walk)->value;
		Node<T> *tmp = *walk;
		*walk = (*walk)->left;
		delete tmp;
	} else if ((*indirect)->right) {
		// look at the right child if the left doesn't exist
		parents[parents_index++] = indirect;
		Node<T> **walk = &(*indirect)->right;
		while ((*walk)->left) {
			parents[parents_index++] = walk;
			walk = &(*walk)->left;
		}
		(*indirect)->value = (*walk)->value;
		Node<T> *tmp = *walk;
		*walk = (*walk)->right;
		delete tmp;
	} else {
		// Otherwise, this is a leaf node
		Node<T> *tmp = *indirect;
		delete tmp;
		*indirect = nullptr;
	}
	//...
``` 


The last step is identical to the one in insertion, we need to update the heights of the nodes we traversed to get to both the node we want to delete, and the node we used to replace the deleted node. We have both of these sets of nodes in our ```parents``` array. Note that unlike insertion, we are not guarenteed that at most one rotation is applied to the tree.

```cpp
	//...
	// update the new heights, applying rotations if necessary
	for (size_t i = 0; i < parents_index; ++i) {
		util::check_for_rotations(parents[parents_index - 1 - i]);
	}
	return root;
} // end of Erase()
```

# Verification

We will want to write some unit tests to make sure our AVL tree actually works. We will need to define some functions to check that our data fields and height fields are correct. 

## Traversal

Let's first define the standard traversal functions so that we can at least do a cursory evaluation of the trees. We pass a container object that implements ```void push_back(typename Node<T>::value_type)``` (Many of the STL containers, such as ```std::vector<T>``` and ```std::list<T>```, implement this function). If you need a refresher on recursion, I learned it from articles like [this one from IBM](https://www.hackerearth.com/practice/basic-programming/recursion/recursion-and-backtracking/tutorial/).

```cpp
template <typename T, typename Container>
void PostOrderTraversal(Node<T> *root, Container &container)
{
	if (!root) return;
	if (root->left) PostOrderTraversal(root->left, container);
	if (root->right) PostOrderTraversal(root->right, container);
	container.push_back(root->value);
}

template <typename T, typename Container>
void InOrderTraversal(Node<T> *root, Container &container)
{
	if (!root) return;
	if (root->left) InOrderTraversal(root->left, container);
	container.push_back(root->value);
	if (root->right) InOrderTraversal(root->right, container);
}

template <typename T, typename Container>
void PreOrderTraversal(Node<T> *root, Container &container)
{
	if (!root) return;
	container.push_back(root->value);
	if (root->left) PreOrderTraversal(root->left, container);
	if (root->right) PreOrderTraversal(root->right, container);
}
```

## Visualizing the Tree

Now we can pass an ```std::vector<T>``` to one of these functions and manually compare the output to some manually created trees. I have written a small command line tool to help visualize trees. In the AVL directory of the source code, run the following: 
```
make display
./display
```

An insert query is created by typing 'i' and then a series of space seperated integers to insert:
```
i 23 -43 0 234 78
```
A delete query is created by typing 'd' followed a series of space seperated integers to delete:
```
e 23 78 34
```
A display query is produced by inputing 'd', which prints the tree. Inputing 'q' exits the program. Here is the output of an example:
```
i 23 -43 0 234 78
e 23 78 34
Value 34 was not found
d
----------------------------------------------------------------
                               0                                
             -43                             234                
----------------------------------------------------------------
q
```

This tool can only do a cursory analysis on smaller trees. We will need to define more verification functions.

## Checking Height and BF

Let's write a recursive function that traverses the tree and checks to make sure heights are correct.

```cpp
// updates heights as it moves along
// returns true if heights are correct
template <typename T>
bool AreHeightsCorrect(Node<T> *root)
{
	bool correct = true;
	if (root->left) correct = correct && IsTreeBalanced(root->left);
	if (root->right) correct = correct && IsTreeBalanced(root->right);
	size_t height = root->height;
	util::update_height(root);
	return (height == root->height && correct);
}
```

Let's write another one that makes sure the tree is balanced 
```cpp
// returns true if balanced false o.w.
template <typename T>
bool IsTreeBalanced(Node<T> *root)
{
	bool balanced = true;
	if (root->left) balanced = balanced && IsTreeBalanced(root->left);
	if (root->right) balanced = balanced && IsTreeBalanced(root->right);
	int bf = util::get_balance_factor(root);
	return (bf <= 1 && bf >= -1 && balanced);
}
```

## Writing Unit Tests

You can look at the unit tests in test.cc. They use the [catch framework](https://github.com/catchorg/Catch2). The tests are by no means extensive, but they are a good start. You can add more edge cases if you want and run them with:
```
make test
```

Here is an example of what one of the tests look like.

```cpp
TEST_CASE("Part1 Tree methods", "[AVL]") {
	// tests...
	SECTION("Insertion") {
		vector<int> vec {}; 
		vec.reserve(9050);
		for (size_t i = 0; i < 50; ++i) 
		  for (int j = -90; j <= 90; ++j) 
			vec.push_back(j);
		std::sort(vec.begin(), vec.end());
		vector<int> inorder_traversal {}; 
		InOrderTraversal(root, inorder_traversal);
		REQUIRE(vec == inorder_traversal);
	}
	// more tests...
```

# Issues with our current tree

We would perhaps like to, as previously described, unify our find and erase functions, and allow iteration through the tree. We cannot do this with our current tree structure, as we will need to be able to traverse up the tree. Let's do that [next](https://hiimkarl.github.io//Learning-AVL-Trees-3/).

* [AVL 1](https://hiimkarl.github.io//Learning-AVL-Trees-1/)
* [AVL 2](https://hiimkarl.github.io//Learning-AVL-Trees-2/)
* [AVL 3](https://hiimkarl.github.io//Learning-AVL-Trees-3/)
* [AVL 4](https://hiimkarl.github.io//Learning-AVL-Trees-4/)
