---
layout: single
title: "AVL Trees -- 3/4: Allowing Upwards Traversal"
---

# Parent Pointer

We can add a pointer to our node's parent in its definition. The root node's parent pointer will point to null. Here's our new node definition:

```cpp
template <typename T>
struct Node {
	typedef T value_type;
	Node(value_type value)
		: value(value), left(nullptr), right(nullptr), parent(nullptr), height(0) 
		{}
	value_type value;
	size_t height;
	Node *left;
	Node *right;
	Node *parent;
};
```

# Insertion
We no longer need to keep track of the path we traversed to get to the place we are inserting the node. We can define a function that grabs the address of the pointer to the node we are examining.

```cpp
template <typename T>
Node<T> *Insert(Node<T> *root, typename Node<T>::value_type val) 
{
	// create a new node to insert into the tree
	Node<T> *new_node = new Node<T>(val);

	// if root is null the new node becomes the root node
	if (!root) return new_node;

	// find out where to put the new node
	// keep track of the route moved into the tree
	Node<T> **indirect = &root;
	while (*indirect) {
		// update the parent of the new_node
		new_node->parent = *indirect;
		if ((*indirect)->value < val) indirect = &(*indirect)->right;
		else indirect = &(*indirect)->left;
	}

	// place the new node into tree
	*indirect = new_node;
	// update the height of the tree, applying rotations if necessary
	while (new_node->parent) {
		util::check_for_rotations(util::pointer_to_node(new_node));
		new_node = new_node->parent;
	}

	// need to the deal with the root node seperately
	indirect = &root;
	util::check_for_rotations(indirect);
	return root;
}
```

Here is the function that grabs the adress of the pointer to a node, provided that the node is not the root. Notice that we have to compare the node to it's parent's children _pointers_, instead of their values, since we can have the case where a node's children both have the exact same value as its parent (because of rotations).

```cpp
// assuming node is not null and it has a parent
template <typename T>
Node<T> **pointer_to_node(Node<T> *node)
{
	Node<T> *parent = node->parent;
	if (parent->left == node) return &(parent->left);
	else return &(parent->right);
} 

```

# Erasing Nodes

As you might imagine, the ```Node<T> Find(typename Node<T>::value_type val)``` remains unchanged. However, we can now pass a pointer to the node we want to erase, allowing us to use ```Node<T> Find(typename Node<T>::value_type val)``` to find a node for the value we want to delete, and verify that it exists, before passing it to erase.

We will define a function called ```void delete_replacement(Node<T> **replacement, Node<T> *dangling_branch)``` to clean up the (replacement) node we are removing. Notice that we have to pass it the dangling branch since it may be the left/right child, depending on which way we walked to find the replacement.

```cpp
template <typename T>
Node<T> *Erase(Node<T> *target) 
{
	Node<T> *walk;
	// find a replacement 
	if (target->left) {
		walk = target->left;
		while (walk->right) 
			walk = walk->right;
		// replace the value and then delete replacement node
		target->value = walk->value;
		delete_replacement(&walk, walk->left);
	} else if (target->right) {
		walk = target->right;
		while (walk->left) 
			walk = walk->left;
		target->value = walk->value;
		delete_replacement(&walk, walk->right);
	} else if (!target->parent) {
		// if the tree is just the single node, return null
		delete target;
		return nullptr;
	} else {
		walk = target;
		delete_replacement(&walk, (Node<T> *)nullptr);
	}

	// update the new heights, applying rotations if necessary
	// traverse until the root node is met
	while (walk->parent) {
		util::check_for_rotations(util::pointer_to_node(walk));
		walk = walk->parent;
	}

	// check if the root node requires a rotation
	Node<T> **root_indirect = &walk;
	util::check_for_rotations(root_indirect);
	return walk;
}
```

This is what our ```delete_replacement``` function looks like:

```cpp
template <typename T>
void delete_replacement(Node<T> **replacement, Node<T> *dangling_branch)
{
	Node<T> **node_ptr = util::pointer_to_node(*replacement);
	Node<T> *parent = (*replacement)->parent;
	(*node_ptr) = dangling_branch;
	if (dangling_branch) dangling_branch->parent = parent;
	delete (*replacement);
	*replacement = parent;
}
```

# Iteration

Let's first implement a forwards iterator. What I mean by that is a function ```Begin(Node<T>)``` will create an iterator that points to the beginning of the tree, while the ```End(Node<T>)``` function will produce an iterator that points to a position __past__ the last node.

## Definition
Our iterator definition only needs to contain the underlying pointer, which we will make private, as the purpose of having an iterator is to hide it. We will want access to the data of the node that the iterator is pointing to, as a reference. We also want to increment and decrement the iterator (but we will not implement arithmetic operations). We can express these functions cleanly with C++ [operator overloading](http://en.cppreference.com/w/cpp/language/operators). 

```cpp
template <typename T>
struct Iterator {
	typedef T value_type;
	Iterator(Node<T> *ptr): ptr(ptr) {}
	Iterator &operator++();
	Iterator &operator--();
	Iterator operator++(int);
	Iterator operator--(int);
	value_type &operator*();
	bool operator==(Iterator const &iterator);
	bool operator!=(Iterator const &iterator) { return !(*this == iterator);}
private:
	Node<T> *ptr;
};
```

## Grabbing the begin and end iterators
We can find the beginning (or the "least value") of the tree if we traverse as far left as possible from the root of the tree. The "end" iterator points to the past-the-last node, which we will for simplicity, just be NULL. This has the consequence of not allowing backwards iteration from the "end" node.

```cpp
template <typename T>
Iterator<T> Begin(Node<T> *root)
{
	Iterator<T> it {root};
	move_it_leftmost(&it);
	return it;
}
```

**Potential Optimization**: Notice that it requires O(log(n)) time to grab the beginning node. If we were instead wrapping everything in a tree structure, we could maintain a pointer to the beginning node. We would have to update it (a simple comparison) on every insertion/deletion.

```cpp
// Get the ending iterator, the parameter doesn't do anything 
// but provide type information
template <typename T>
Iterator<T> End(Node<T> *root)
{
	return Iterator<T> {nullptr};
}
```

## Access and equality

These functions are really straight forward. Notice that for iterators to be equivalent, the node they point to must be equivalent, not just their values.

```cpp
template <typename T>
bool Iterator<T>::operator==(Iterator<T> const& iterator)
{
	return ptr == iterator.ptr;
}
```

```cpp
template <typename T>
typename Iterator<T>::value_type &Iterator<T>::operator*() 
{ 
	return ptr->value; 
}
```

## Increment and Decrement

This will require a little bit of thinking. If we are incrementing the node, then it should be obvious that we should move to the right child if it exists. There is never a situation where we move to the left child (since it is always ordered before). The key observation that has to be made is that if the node we are stepping from is itself a right child, then its parent is lower in the order, so we will have already iterated through it. Thus, we should jump upwards until our current node is no longer a right child. If the node is a left child, its parent is the next node in the order we are iterating through.

Here is what the prefix operator looks like:
```cpp
// Assumes *this is not the end iterator
template <typename T>
Iterator<T> &Iterator<T>::operator++()
{
	// We can perform a check here for the end iterator (ptr is NULL)
	// and throw an exception if found
	if (ptr->right) {
		ptr = ptr->right;
		move_it_leftmost(this);
	} else {
		while (ptr->parent && !iterator_is_left_child(this)) ptr = ptr->parent;
		ptr = ptr->parent;
	}
	return *this;
}
```

I defined a couple of small utility functions to make the code more readable. These functions must be [friends](http://en.cppreference.com/w/cpp/language/friend) with ```Iterator``` because they access the underling pointer.

```cpp
template <typename T> struct Iterator {
	// stuff...	
	template <typename Y>
	friend inline void move_it_leftmost(Iterator<Y> *iterator)
	{
		while (iterator->ptr->left) iterator->ptr = iterator->ptr->left;
	}
	template <typename Y>
	friend inline void move_it_rightmost(Iterator<Y> *iterator)
	{
		while (iterator->ptr->right) iterator->ptr = iterator->ptr->right;
	}
	template <typename Y>
	friend bool iterator_is_left_child(Iterator<Y> *iterator)
	{
		if (iterator->ptr == iterator->ptr->parent->left) return true;
		return false;
	}
	// more stuff...
};
```

Here is the postfix operator, which behaves exactly as if x was an int and you wrote ```x++```. C++ differentiates their definition expressions with type information passed as a parameter.
```cpp
template <typename T>
Iterator<T> Iterator<T>::operator++(int)
{
	Iterator<T> tmp = *this;
	++*this;
	return tmp;
}
```

The decrement operator does exactly the opposite of what the increment operator does. Notice that there is no convient way to grab an iterator to the _last_ element of the tree, as we can perform this operation on the end iterator, so we will need to implement a reverse iterator. Containers in the C++ standard library are implemented with a forwards and reverse iterator (however, unlike our tree, you can decrement STL container end iterators).  

```cpp
template <typename T>
Iterator<T> &Iterator<T>::operator--()
{
	// We can perform a check for the begin iterator (no left-child and is a left child)
	// and throw an exception if found
	if (ptr->left) {
		ptr = ptr->left;
		move_it_rightmost(this);
	} else {
		while (iterator_is_left_child(this)) ptr = ptr->parent;
		ptr = ptr->parent;
	}
	return *this;
}
```

Notice that for any given increment operation, at best the operation runs in O(1), but at worst it requires O(log(n)) time. Combining with decrement, you could get stuck with a lot of really expensive iterator operations. 

However, the runtime of just iterating through the entire tree without performing any decrements is O(n), so the ammortized runtime of each increment is O(1). An easy way to prove this is using the accounting method (you can find it in CH18 of CLRS). Assuming following a pointer to another node requires $1, it is easy to see you only need to assign $2 to each node to cover the cost of iterating through the tree.

# Conclusion

Next up, we conclude the series by talking about additional features we can add and we can test how efficient the tree is by performing a benchmark against std::set.

* [AVL 1](https://hiimkarl.github.io//Learning-AVL-Trees-1/)
* [AVL 2](https://hiimkarl.github.io//Learning-AVL-Trees-2/)
* [AVL 3](https://hiimkarl.github.io//Learning-AVL-Trees-3/)