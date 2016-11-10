# Printing a linked list in gdb 
## (A glimpse into the GDB Python APIs)

Does the following look like something you have done before?

~~~
(gdb) p *s_list_head->next->next->next
$5 = {
  next = 0x100101140,
  [...]
}
~~~

There's a better way! [gdb 7][1] (dating all the way back to 2009) introduced support for python scripting. This allows one to script gdb operations using python!

All you need is a gdb toolchain compiled with this support. This can be easily checked by looking that `--with-python` is in the configuration:

~~~
gdb --configuration | grep python
             --with-python=/System/Library/Frameworks/Python.framework/Versions/2.7
~~~

In this article we walk through a very simple use case to script the printing of all the nodes in a linked list.

# The Example C Code

```c
#include <stdio.h>
#include <stdlib.h>

//
// A boring singly linked list where each node holds 1 random number
//

typedef struct ListNode {
  struct ListNode *next;
  uint32_t random_value;
} ListNode;

static ListNode *s_list_head = NULL;

void list_add_random_value(uint32_t random_value) {
  ListNode *node_to_add = malloc(sizeof(ListNode));
  if (node_to_add == NULL) {
    printf("unexpected malloc failure\n");
    return;
  }

  *node_to_add = (ListNode) {
    .next = NULL,
    .random_value = random_value,
  };

  // Add entry to the front of the list
  node_to_add->next = s_list_head;
  s_list_head = node_to_add;
}

static void free_list(void) {
  while (s_list_head != NULL) {
    ListNode *temp = s_list_head->next;
    free(s_list_head);
    s_list_head = temp;
  }
}

static void populate_example_list(void) {
  for (int i = 0; i < 5; i++) {
    list_add_random_value(rand());
  }
}

int main(void) {
  populate_example_list();
  free_list();
  return 0;
}
```

# Python script

You can find documentation about the GDB Python API starting [here](https://sourceware.org/gdb/onlinedocs/gdb/Python.html#Python). In the following script we show how one could go about walking through the `s_list_head` node from above.

```python
import uuid
import gdb

class ListNodePrinter(gdb.Command):
   """Prints the ListNode from our example in a nice format!"""

   def __init__(self):
       super(ListNodePrinter, self).__init__("walklist", gdb.COMMAND_USER)

   def invoke(self, args, from_tty):
       # You can pass args here so this routine could actually evaluate different
       # variables at runtime
       print "Args Passed: %s" % args

       # Let's walk through the list starting with the head
       #
       # We can access value info by looking at:
       #  https://sourceware.org/gdb/onlinedocs/gdb/Values-From-Inferior.html#Values-From-Inferior
       node_ptr = gdb.parse_and_eval("s_list_head")

       count = 0
       while node_ptr != 0:
           print "%d: Addr: 0x%x, random value: %s" % \
               (count, node_ptr, int(node_ptr['random_value']))
           node_ptr = node_ptr['next']
           count += 1
       print "Found %d nodes" % count

ListNodePrinter()
```

# Trying it out

#### Copy & Paste the C snippet from above into `test.c` and compile it with debug symbols:

```
gcc -std=c99 -g test.c -o test
```

#### Launch the application in GDB

``` 
gdb ./test 
```

#### Add a breakpoint right before the list gets freed: 

``` 
(gdb) break free_list
Breakpoint 1 at 0x100000f08: file test.c, line 34.
```

#### Copy & Paste the Python snippet into `list_dump.py` and source it from within gdb

```
(gdb) source list_printer.py
```

#### Run the application halting at the breakpoint

```
(gdb) r
Starting program: test

Breakpoint 1, free_list () at test.c:34
34	  while (s_list_head != NULL) {
```

#### Experiment with manually printing the list contents

```
(gdb) print s_list_head
$1 = (ListNode *) 0x1002002b0
(gdb) print *s_list_head
$2 = {
  next = 0x1002002a0,
  random_value = 1144108930
}
(gdb) print *(s_list_head->next)
$3 = {
  next = 0x100200290,
  random_value = 984943658
}
(gdb) print *(s_list_head->next->next)
$4 = {
  next = 0x1002000b0,
  random_value = 1622650073
}
```

#### Now observe how easily we can look at everything with our new python function!

```
(gdb) walklist
Args Passed:
0: Addr: 0x1002002b0, random value: 1144108930
1: Addr: 0x1002002a0, random value: 984943658
2: Addr: 0x100200290, random value: 1622650073
3: Addr: 0x1002000b0, random value: 282475249
4: Addr: 0x1002000a0, random value: 16807
Found 5 nodes
```


[1]: https://www.gnu.org/software/gdb/news/