## redis源码—简单动态字符串SDS

SDS是redis自己构建的简单动态字符串。</br>
源码的位置是在sds.h中。（Redis版本为4.0.8）
### SDS的原理
- 原理很简单，头部中记录了已经使用的长度和总的分配长度（之前的版本为记录剩余长度）。</br>
4.0版本针对不同的字符串长度使用了不同的结构体。</br>
比如长度小于32的字符串，则会使用sdshdr5结构体。
sdshdr8、sdshdr16、sdshdr32、sdshdr64结构一样。
- 与c语言中的字符串相比：
- 获取字符串长度的复杂度</br>
c语言中的字符串并不记录自身的长度，要遍历获取，所以时间复杂度(n)。</br>
SDS在len的属性中记录了SDS的长度。时间复杂度从O(n)降到O(1)。
- 杜绝缓冲区溢出
这也是c语言中字符串记录长度导致的。SDS是动态分配空间的，不会出现缓冲区溢出情况。
- 减少修改字符串时带来的内存重分配次数
想追加字符串时不需要像单纯C语言那样重新开辟一块空间然后将原字符串和追加内容一起拷贝过去，而是直接将其添加到SDS未分配的空间中，当然，遇到剩余未分配空间不足的情况则需要进行扩容。
- 二进制安全。

源码如下：
```c
// 类型别名，指向sdshdr结构体中的buf属性，也就是实际的字符串的指针
// 注意将其与大写SDS区分
// SDS泛指的Redis设计的动态字符串结构(结构体sdshdr + 实际字符串sds)
typedef char *sds;

/* Note: sdshdr5 is never used, we just access the flags byte directly.
* However is here to document the layout of type 5 SDS strings. */
// __attribute__是为了增强编译器检查
// __packed__则是告诉编译器则可能少的分配内存
struct __attribute__ ((__packed__)) sdshdr5 {
// flags既是标记了头部类型，同时也记录了字符串的长度
// 共8位，flags用前5位记录字符串长度（小于32=1<<5），后3位作为标志
unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
// 字符串的长度，即已经使用的buf长度
uint8_t len; /* used */
//为buf分配的总的长度，之前的版本记录的是free（剩下的长度）
uint8_t alloc; /* excluding the header and null terminator */
// 新增属性，记录该结构体的实际类型
unsigned char flags; /* 3 lsb of type, 5 unused bits */
// 柔性数组，为结构体分配内存的时候顺带分配，作为字符串的实际存储内存
// 由于buf不占内存，所以buf的地址就是结构体尾部的地址，也是实际字符串开始的地址
char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
uint16_t len; /* used */
uint16_t alloc; /* excluding the header and null terminator */
unsigned char flags; /* 3 lsb of type, 5 unused bits */
char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
uint32_t len; /* used */
uint32_t alloc; /* excluding the header and null terminator */
unsigned char flags; /* 3 lsb of type, 5 unused bits */
char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
uint64_t len; /* used */
uint64_t alloc; /* excluding the header and null terminator */
unsigned char flags; /* 3 lsb of type, 5 unused bits */
char buf[];
};
```
新版本定义了5种数据类型，会根据不同的字符串长度来分配：
源码如下：
```c
/**
根据字符串的长度决定要使用的结构体的类型
1、string_size 用于初始化的字符串的长度
2、返回值：要使用的sdshdr类型有五种。只有SDS_TYPE_5比较特殊，只记录了flags和buf
**/
static inline char sdsReqType(size_t string_size) {
if (string_size < 1<<5)
return SDS_TYPE_5;
if (string_size < 1<<8)
return SDS_TYPE_8;
if (string_size < 1<<16)
return SDS_TYPE_16;
#if (LONG_MAX == LLONG_MAX)
if (string_size < 1ll<<32)
return SDS_TYPE_32;
#endif
return SDS_TYPE_64;
}
```
### 创建SDS
创建SDS有四种方法：
```c
sds sdsnewlen(const void *init, size_t initlen);
sds sdsnew(const char *init);
sds sdsempty(void);
sds sdsdup(const sds s);
```
其他函数实际上也是调用sdsnewlen()。
源码如下：
```c
sds sdsnewlen(const void *init, size_t initlen) {
void *sh;
sds s;
char type = sdsReqType(initlen);
/* Empty strings are usually created in order to append. Use type 8
* since type 5 is not good at this. */
if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
int hdrlen = sdsHdrSize(type);
unsigned char *fp; /* flags pointer. */

sh = s_malloc(hdrlen+initlen+1);
if (!init)
memset(sh, 0, hdrlen+initlen+1);
if (sh == NULL) return NULL;
s = (char*)sh+hdrlen;
fp = ((unsigned char*)s)-1;
switch(type) {
case SDS_TYPE_5: {
*fp = type | (initlen << SDS_TYPE_BITS);
break;
}
case SDS_TYPE_8: {
SDS_HDR_VAR(8,s);
sh->len = initlen;
sh->alloc = initlen;
*fp = type;
break;
}
case SDS_TYPE_16: {
SDS_HDR_VAR(16,s);
sh->len = initlen;
sh->alloc = initlen;
*fp = type;
break;
}
case SDS_TYPE_32: {
SDS_HDR_VAR(32,s);
sh->len = initlen;
sh->alloc = initlen;
*fp = type;
break;
}
case SDS_TYPE_64: {
SDS_HDR_VAR(64,s);
sh->len = initlen;
sh->alloc = initlen;
*fp = type;
break;
}
}
if (initlen && init)
memcpy(s, init, initlen);
s[initlen] = '\0';
return s;
}
```
