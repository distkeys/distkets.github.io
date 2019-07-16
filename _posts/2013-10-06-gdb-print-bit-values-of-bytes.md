---
layout: post
title: "GDB - Print Bit values of bytes"
date: 2013-10-06 21:06
comments: true
categories: Debug
tags: '2013'
---
{% include toc %}

## Print bit values in a byte
Recently, I have been working on interesting piece of code whose crux is to create a array of pointer addresses.
Each entry in this array is address pointing to memory location.

For example<br>
Container array contains char addresses. Here, 100 is memory address where char value resides.

100 - 1000 - 2000

Address 100<br>
v | a | i | b | h | a | v | \0

Sometimes char data type is used as a package of 8 bits not as a valid char value.<br>

### Code snippet

{% highlight c linenos %}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main()
{
    char **container = (char **)malloc(10 * sizeof(char*));
    char **start = container;
    char *node;

    char name[] = "Vaibhav";
    int i = 0;

    if (container == NULL) {
	return 0;
    }

    for (i = 0; i <= 2; i++) {
	node = (char *)malloc(10 * sizeof(char));
	memcpy(node, &name, strlen(name) + 1); 
	*container = node;
	container++;
    }
    *container = NULL;

    while (*start != NULL) {
	printf("%s\n", *start);
	start++;
    }

    return 0;
}
{% endhighlight %}

<br>
Focusing on following code section

~~~
for (i = 0; i < = 2; i++) {
    node = (char *)malloc(10 * sizeof(char));
    memcpy(node, &name, strlen(name) + 1); 
    *container = node;
    container++;
}
~~~
{:lang="c"}

<br>In this section, a memory of 10 chars is being allocated, initialized and finally assigned to container array.
<br>Lets observer, if we have set the right information in each char bit.

*Compile code using for GDB*
> gcc -g fileName.c

<br>

{% highlight c linenos %}
(gdb) l
16      }
17	
18	    for (i = 0; i < = 2; i++) {
19	        node = (char *)malloc(10 * sizeof(char));
20	        memcpy(node, &name, strlen(name) + 1);
21	        *container = node;
22	        container++;
23	    }
24	    *container = NULL;
25	
(gdb) ptype node
type = char *
(gdb) p node
$1 = 0x1001000e0 "Vaibhav"
(gdb) x/8bb node
0x1001000e0:	0x56	0x61	0x69	0x62	0x68	0x61	0x76	0x00
(gdb) x/8ub node
0x1001000e0:	86	97	105	98	104	97	118	0
(gdb) x/8tb node
0x1001000e0:	01010110	01100001	01101001	01100010	01101000	01100001	01110110	00000000
{% endhighlight %}
