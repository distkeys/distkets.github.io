---
layout: post
title: "Virtual Memory - Part 2"
date: 2015-01-02 20:25
comments: true
categories:
 - Operating Systems

tags:
- '2015'
---

This is continuation of part series post of [Virtual Memory - Part 1](http://distkeys.com/operating%20systems/2015/01/02/virtual-memory-1.html)


{% include toc %}

<br>
## Paging

In paging, we divide physical memory into fixed-size blocks, typically 4-16 KB. So, in theory, it could be understood as 0-100 address range is 1 page, 100-200 addresses is another page and so on. So we now have two components Page number and offset. The offset value is used to go to a specific address.

Page Map or Page Table is simply a translation of Virtual address to a physical address.

> Virtual address = Virtual page number + offset bits
>
>Physical address = Physical page number + offset bits

<br>
Based on the size of the page, a number of bits required to represent an individual address within the page can change.

Lets say if page size if 4K i.e. one page can have addresses from 0-4000 then for offset we need 12 bits as 2<sup>12</sup> = 4096. For page size of 16K i.e. one page can have addresses from 0-16000 we need 14 bits as 2<sup>14</sup> = 16384.


On a high level, simple Page Map design may look like this.


<img src="/images/pagemap.png" width="500" align="middle">

- Resident bit 'R' is 1 when a page is in Physical Memory if 0 then a Page fault
- Physical page number (PPN)
- Dirty bit D

<br>
**Example**

In this example, a Page can hold 256 addresses so Offset needs 8 bits. We have a total of 16 virtual pages, so Index bits needs 4 bits. For main memory, we have space for only 8 physical pages which translate to 3 bits for Index and 8 bits for Offset.

<img src="/images/examplepagemap.png" width="400" align="middle">


**Virtual Addresses**

- Page Size - 256 Bytes 2<sup>8</sup>
- Total Virtual Pages - 16 (2<sup>4</sup>)
- Total Bits 12 = 4 VPN + 8 Offset Bits

<br>
**Physical Addresses**

- Page Size - 256 Bytes (2<sup>8</sup>)
- Total Physical Pages - 8 (2<sup>3</sup>)
- Total Bits 11 = 3 (PPN) + 8 (Offset Bits)

Let's translate Virtual Address [0x2C8] to Physical Address


$$0X2C8 = \underbrace{0010}_\text{PageMap Index} \ \underbrace{1100 \ 1000}_\text{Offset Bits}$$

$$0X2C8 = \underbrace{2}_\text{PageMap Index} \ \underbrace{200}_\text{Offset Bits}$$

$$0X2C8 = \underbrace{0X2}_\text{PageMap Index} \ \underbrace{0XC8}_\text{Offset Bits}$$

$$Page Map\lbrack2\rbrack = \lbrack D = 0, R = 1, PPN\_Entry = 4\rbrack$$

$$Index \ bit \ PPN[4] = 0X400$$

$$Offset \ from  (3) = 0XC8$$

$$ Final Physical Address = 0X400 + 0XC8 = 0X4C8$$


Offset value never changes from VPN to PPN(Physical Page Number) as page size is constant across the system.


<br><br>
## Page Fault


In the case of Page Fault i.e., Page does not exist in the Page Map table as well as in Main Memory. In this case, we need to make room for a new page that means we need to evict an existing page entry from page map table as well as a page from main memory.

Page fault can also occur when entry exists in Page Maps table, but its resident bit is set to 0, which means the page is not present in main memory but will be available on disk.

We have two cases

- When entry does not exist in Page Map entry
- When entry does exist in Page Map entry but does not exist in main memory

<br>
Following is the sequence diagram for the above two cases.


![center-aligned-image](/images/pagefaultseq.png)
