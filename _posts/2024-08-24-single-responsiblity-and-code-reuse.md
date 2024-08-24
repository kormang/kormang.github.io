
_"The more we build things by small pieces, the more the same underlying resources can be reused, making the whole system simpler to use."_ - "FreeSWITCH" 1.8 by Anthony Minessale II and Giovanni Maruzzelli.

### Relation between single responsibility and code reuse

**The relationship between single responsibility (and separation of concerns) on one side, and code reuse on the other, is the single most important concept to grasp for a professional software engineer.**

Let's start by explaining the **Single Responsibility Principle**. This principle, which is the first one in the **SOLID** principles introduced by Robert C. Martin, says "A class should have only one reason to change". This might not be the clearest explanation because it also applies to other parts of code, not just classes. He also put it in another, possible better way: "A module should be responsible to one, and only one, actor."

Now, let's look at the **Separation of Concerns** principle. This idea was first mentioned by Edsger W. Dijkstra in 1974 and was later described by Chris Reade in a 1989 book about functional programming. You can read more about this in the Wikipedia page called [Separation of Concerns](https://en.wikipedia.org/wiki/Separation_of_concerns).

As Mr. Reade points out, the things we need to pay attention to can vary depending on the programming language we're using, kind of like how different design patterns make sense in different languages. This brings us to concerns that are specific to a certain language.

Now, let's talk about one specific concern, which is memory management.

_(If you're not familiar with the C programming language, try your best to understand the next part. If it's too difficult, you can skip it.)_

Think about this C language function that finds the most repeated number in a sequence:

```C
int most_frequent(int numbers[], int length) {
  ...
}
```

To implement this function in C, we need to allocate memory for data structures that will hold counts for each number in the array, and we need to deallocate it (free it), when no longer needed.

In other languages, like C++, this responsibility can be put on the compiler and standard library, and if we're using language like JavaScript that has garbage collector, garbage collector will take care of memory deallocation. That means that depending on the language, certain responsibilities can be separated from our main logic.

This is not 0 or 1 case, true or false case, where responsibility is separated or not.
As those who know C can confirm, on one end of the spectrum is allocating data structure and initializing it in some loop right there in the function, and then releasing it at the end, and on the other is having some of that done by special destructor functions.

Let's look at the full example.

```C
struct bucket {
  int key;
  int value;
  struct bucket* next;
};

int most_frequent(int numbers[], int length) {
  struct bucket* hash_table = calloc(257, sizeof(struct bucket));

  // Run the algorithm and obtain the result.
  // Count all occurences.
  for (int i = 0; i < length; ++i) {
  	int hash = (unsigned int)(numbers[i]) % 257;
  	if (hash_table[hash].value == 0 || hash_table[hash].key == numbers[i]) {
  		hash_table[hash].key = numbers[i];
  		hash_table[hash].value++;
  	} else {
  		struct bucket* prev_b = &hash_table[hash];
  		struct bucket* b = hash_table[hash].next;
  		while (b) {
  			if (b->key == numbers[i]) {
  				b->value++;
  				break;
  			}
  			prev_b = b;
  			b = b->next;
  		}

  		if (!b) {
  			prev_b->next = calloc(1, sizeof(struct bucket));
  			b = prev_b->next;
  			b->key = numbers[i];
  			b->value = 1;
  		}
  	}
  }

  // Find max count.
  int maxCount = 0;
  int result = 0;
  for (int i = 0; i < 257; ++i) {
  	if (hash_table[i].value > 0) {
  		struct bucket* b = &hash_table[i];
  		while (b) {
  			if (b->value > maxCount) {
  				maxCount = b->value;
  				result = b->key;
  			}
  			b = b->next;
  		}
  	}
  }


  // Free hash table.
  for (int i = 0; i < 257; ++i) {
    struct bucket* b = hash_table[i].next;
    while (b) {
      struct bucket* t = b;
      b = b->next;
      free(t);
    }
  }
  free(hash_table);

  return result;
}
```

This code is not ideal in many aspects, but we can ignore small things like using a hard-coded length for the hash table (257), and things like that because it basically works, and we want to focus on something else.

Here, everything is mixed up. We have memory allocation/deallocation responsibility (memory management), we have hash table management and hash table traversal responsibility, and finally, we have our main algorithm. So we have at least three different responsibilities in one function.

Still, things can be much better, even in C. We could use a function that allocates and initializes these data structures in one go and releases all the memory using another function call. The management of the hash table state and traversal can also be extracted.

We can extract following functions for hash table:
* `hash_table_init` - allocates and initializes hash table
* `hash_table_get_bucket` - returns bucket corresponding to provided key
* `hash_table_set` - set provided value under the provided key
* `hash_table_iterator` - returns new iterator over the non-empty buckets
* `hash_table_it_next` - moves on to the next bucket and returns pointer to it, enabling iteration

Following block of code contains details of implementation for those functions, and can be skipped.
The block of code after this one, will demonstrate how simple implementation of `most_frequent` can be.

```C
struct bucket {
  int key;
  int value;
  int direct_full:1;
  struct bucket* next;
};

struct hash_table {
  int size;
  // Buckets will be dynamically allocated.
  // struct bucket buckets[];
};

#define HASH_TABLE_GET_BUCKETS(hash_table) (struct bucket*)((hash_table) + 1)

struct hash_table* hash_table_init(int size) {
  struct hash_table* hash_table = calloc(1, sizeof(struct hash_table) + size * sizeof(struct bucket));
  hash_table->size = size;
  return hash_table;
}

struct bucket* hash_table_get_bucket(struct hash_table* hash_table, int key) {
  int hash = (unsigned int)(key) % hash_table->size;
  struct bucket* buckets = HASH_TABLE_GET_BUCKETS(hash_table);
  struct bucket* b = &buckets[hash];

  if (b->key == key && b->direct_full) {
    return b;
  }

  while (b) {
    if (b->key == key) {
      break;
    }
    b = b->next;
  }

  return b;
}

void hash_table_set(struct hash_table* hash_table, int key, int value) {
  struct bucket* buckets = HASH_TABLE_GET_BUCKETS(hash_table);

  int hash = (unsigned int)(key) % hash_table->size;
  struct bucket* b_prev = &buckets[hash];

  if (!b_prev->direct_full) {
    b_prev->key = key;
    b_prev->value = value;
    b_prev->direct_full = 1;
  }

  struct bucket* b = b_prev->next;

  while (b) {
    if (b_prev->key == key) {
      b_prev->value = value;
      return;
    }
    b_prev = b;
    b = b->next;
  }

  b_prev->next = (struct bucket*)calloc(1, sizeof(struct bucket));
  b_prev->next->key = key;
  b_prev->next->value = value;
}

struct hash_table_it {
  struct bucket* current;
  struct bucket* head;
  struct bucket* last;
};

struct hash_table_it hash_table_iterator(struct hash_table* hash_table) {
  struct bucket* buckets = HASH_TABLE_GET_BUCKETS(hash_table);

  struct hash_table_it it = {
    NULL,
    buckets,
    buckets + hash_table->size
  };

  return it;
}

struct bucket* hash_table_it_next(struct hash_table_it* it) {
  if (it->current == NULL) {
    it->current = it->head;
    if (it->current->key) {
      return it->current;
    }
  }

  if (it->current == it->last) {
    return NULL;
  }

  // Find next.
  if (it->current->next) {
    it->current = it->current->next;
    return it->current;
  }

  it->head++;
  while (it->head != it->last && !it->head->key) {
    it->head++;
  }
  it->current = it->head;
  if (it->current == it->last) {
    return NULL;
  }
  return it->current;
}

void hash_table_free(struct hash_table* hash_table) {
  struct bucket* buckets = HASH_TABLE_GET_BUCKETS(hash_table);
  for (int i = 0; i < hash_table->size; ++i) {
    struct bucket* b = buckets[i].next;
    while (b) {
      struct bucket* t = b;
      b = b->next;
      free(t);
    }
  }
  free(hash_table);
}
```

Now implementation of `most_frequent` is super simple and readable. At the same time, we have couple of functions and a reusable data structure that we can use in many other places.

```C
int most_frequent(int numbers[], int length) {
  struct hash_table* hash_table = hash_table_init(257);

  // Run the algorithm and obtain the result.
  // Count all occurrences.
  for (int i = 0; i < length; ++i) {
  	struct bucket* b = hash_table_get_bucket(hash_table, numbers[i]);
    if (b) {
      b->value++;
    } else {
      hash_table_set(hash_table, numbers[i], 1);
    }
  }

  // Find max count.
  int maxCount = 0;
  int result = 0;
  struct hash_table_it it = hash_table_iterator(hash_table);
  struct bucket* bucket;

  while (bucket = hash_table_it_next(&it)) {
    if (bucket->value > maxCount) {
      maxCount = bucket->value;
      result = bucket->key;
    }
  }

  hash_table_free(hash_table);

  return result;
}
```

Here, we haven't gotten rid of this responsibility to free allocated resources; the language doesn't really allow us to do it. However, we did get rid of the responsibility to allocate and free the hash table properly by using `hash_table_init` and `hash_table_free`. For setting a value into the hash table, getting a value, and iterating over elements, we use `hash_table_get_bucket`, `hash_table_set`, `hash_table_iterator`, and `hash_table_it_next`. We can understand the code inside `most_frequent` even without looking at the implementation of all those functions. That's actually the beauty of it — this is why it's cleaner and easier to understand — because **there's less to understand**. Now we don't need to know about the internal structure of the hash table, how to properly compute a hash, how the iteration works internally, and so on. Yet, we still understand how the `most_frequent` function works. The code is cleaner, and we can focus on the algorithm itself. Fewer aspects of the program are mixed in one function.

**The best of all is that we can reuse those hash table functions in different places.** Code that manages hash table structure is extracted and separated from `most_frequent` function, and can be reused in different functions, different parts of the code. That is exactly **the relation between code reuse and the single responsibility principle**.

Another important benefit is that we can now modify the implementation of those hash table functions, make improvements, and possibly optimize them. This way, we can automatically improve all the places where the hash table is used.

We might think that this is obvious and not something new. However, we often end up writing duplicated and closely connected code, rather than creating reusable code with clear boundaries between responsibilities. This often happens because initially, we only need to implement the `most_frequent` function, so we put everything into that one function, thinking it's "easier." But in reality, it becomes more difficult for others to understand. This approach also becomes a foundation for future creation of poor-quality code. When others need a hash table, they might either duplicate our code or create their own implementation, leading to code duplication in both cases. Duplication itself isn't the main problem. The issue is that we can't improve the hash table implementation in one place, making the code harder to maintain. Additionally, each duplicated hash table implementation might have its own bugs. So, if possible, it's a good idea to be proactive and write code with separated concerns, or at the very least, refactor it proactively, and in time.

Do not use reusable code only when it comes in form of a library or framework, written by other people. Write your own mini libraries of reusable parts.

> NOTE: Sometimes a little bit of duplication is OK, more on that later.

For the end, let's see how it would look like in TypeScript:

```TypeScript
function mostFrequent(numbers: number[]): number {
  const hashTable = new Map<number, number>();

  // Run the algorithm and obtain the result.
  // Count all occurrences.
  for (let i = 0; i < numbers.length; ++i) {
    hashTable.set(numbers[i], hashTable.has(numbers[i]) ? hashTable.get(numbers[i])! + 1 : 0)
  }


  // Find max count.
  let maxPair = [-1, 0]
  for (const pair of hashTable.entries()) {
    if (pair[1] > maxPair[1]) {
      maxPair = pair;
    }
  }
  result = maxPair[0]

  return result
}
```

Here, the garbage collector takes care of the helper data structure, and its constructor and methods manage its state. So now the code is absolutely clean — free from all other responsibilities, other than the algorithm itself. Hash table, represented by class `Map` here, is already built-in reusable component, that we often take for granted.
