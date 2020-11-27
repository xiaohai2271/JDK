# System类中的一个native方法源代码解读

首先上方法签名

```java
public static native void arraycopy(Object src,  int  srcPos,
                                    Object dest, int destPos,
                                    int length);
```

至于我为什么会关注到这个方法上面来还带从之前看集合框架的源代码说起。在集合框架中对数组进行操作时，都会直接或者间接的调用此方法。网上查资料说此native方法很快，能带来很高的性能提升。那为什么一个数组拷贝的native方法就能带来高性能呢 ？

想要解决这个问题最好的办法就是去看这个方法是怎么实现的。

于是我开始追踪虚拟机的代码执行，就有了下面的内容.......

> 下载源代码，构建jdk编译调试环境 本篇暂不作介绍

## 0x00 定位native方法的申明位置

此方法位于System类，而System类又处于java.base中的java.lang包下。于是我们可以很容易的找到` src/java.base/share/native/libjava/System.c`文件，在该文件中我们可以看到

![image-20201128005814040](pic/System.arraycopy/image-20201128005814040.png)

这里有native方法的申明。其中关键的就是`(void *)&JVM_ArrayCopy` 这一句。这将是我们寻根溯源的第一个入口。

我们全局搜索可以看到jvm.h中有![image-20201128010348420](pic/System.arraycopy/image-20201128010348420.png)

接下来去jvm.cpp瞅瞅看下有木有实现，然后有了下面这一段

```c++
JVM_ENTRY(void, JVM_ArrayCopy(JNIEnv *env, jclass ignored, jobject src, jint src_pos,
                               jobject dst, jint dst_pos, jint length))
  JVMWrapper("JVM_ArrayCopy");
  // Check if we have null pointers
  if (src == NULL || dst == NULL) {
    THROW(vmSymbols::java_lang_NullPointerException());
  }
  arrayOop s = arrayOop(JNIHandles::resolve_non_null(src));
  arrayOop d = arrayOop(JNIHandles::resolve_non_null(dst));
  assert(oopDesc::is_oop(s), "JVM_ArrayCopy: src not an oop");
  assert(oopDesc::is_oop(d), "JVM_ArrayCopy: dst not an oop");
  // Do copy
  s->klass()->copy_array(s, src_pos, d, dst_pos, length, thread);
JVM_END	
```

简单的看下代码以及所带的少量注释我们可以看出最后一句才是进行复制的关键语句，那么接着往下

通过s的类类型我们找到`klass`类发现有申明`copy_array`方法但是并没有发现实现方法

![](pic/System.arraycopy/image-20201128012027417.png)

搜索发现`src/hotspot/share/oops/objArrayKlass.cpp`,`src/hotspot/share/oops/typeArrayKlass.cpp`中有包含相同方法前面的函数。两函数比较如图

![image-20201128012420520](pic/System.arraycopy/image-20201128012420520.png)

代码如下

- `src/hotspot/share/oops/objArrayKlass.cpp`

```java
void TypeArrayKlass::copy_array(arrayOop s, int src_pos, arrayOop d, int dst_pos, int length, TRAPS) {
  assert(s->is_typeArray(), "must be type array");

   printf("zhenghai ->> src/hotspot/share/oops/typeArrayKlass.cpp:109\n");

    // Check destination type.
  if (!d->is_typeArray()) {
    ResourceMark rm(THREAD);
    stringStream ss;
    if (d->is_objArray()) {
      ss.print("arraycopy: type mismatch: can not copy %s[] into object array[]",
               type2name_tab[ArrayKlass::cast(s->klass())->element_type()]);
    } else {
      ss.print("arraycopy: destination type %s is not an array", d->klass()->external_name());
    }
    THROW_MSG(vmSymbols::java_lang_ArrayStoreException(), ss.as_string());
  }
  if (element_type() != TypeArrayKlass::cast(d->klass())->element_type()) {
    ResourceMark rm(THREAD);
    stringStream ss;
    ss.print("arraycopy: type mismatch: can not copy %s[] into %s[]",
             type2name_tab[ArrayKlass::cast(s->klass())->element_type()],
             type2name_tab[ArrayKlass::cast(d->klass())->element_type()]);
    THROW_MSG(vmSymbols::java_lang_ArrayStoreException(), ss.as_string());
  }

  // Check if all offsets and lengths are non negative.
  if (src_pos < 0 || dst_pos < 0 || length < 0) {
    // Pass specific exception reason.
    ResourceMark rm(THREAD);
    stringStream ss;
    if (src_pos < 0) {
      ss.print("arraycopy: source index %d out of bounds for %s[%d]",
               src_pos, type2name_tab[ArrayKlass::cast(s->klass())->element_type()], s->length());
    } else if (dst_pos < 0) {
      ss.print("arraycopy: destination index %d out of bounds for %s[%d]",
               dst_pos, type2name_tab[ArrayKlass::cast(d->klass())->element_type()], d->length());
    } else {
      ss.print("arraycopy: length %d is negative", length);
    }
    THROW_MSG(vmSymbols::java_lang_ArrayIndexOutOfBoundsException(), ss.as_string());
  }
  // Check if the ranges are valid
  if ((((unsigned int) length + (unsigned int) src_pos) > (unsigned int) s->length()) ||
      (((unsigned int) length + (unsigned int) dst_pos) > (unsigned int) d->length())) {
    // Pass specific exception reason.
    ResourceMark rm(THREAD);
    stringStream ss;
    if (((unsigned int) length + (unsigned int) src_pos) > (unsigned int) s->length()) {
      ss.print("arraycopy: last source index %u out of bounds for %s[%d]",
               (unsigned int) length + (unsigned int) src_pos,
               type2name_tab[ArrayKlass::cast(s->klass())->element_type()], s->length());
    } else {
      ss.print("arraycopy: last destination index %u out of bounds for %s[%d]",
               (unsigned int) length + (unsigned int) dst_pos,
               type2name_tab[ArrayKlass::cast(d->klass())->element_type()], d->length());
    }
    THROW_MSG(vmSymbols::java_lang_ArrayIndexOutOfBoundsException(), ss.as_string());
  }
  // Check zero copy
  if (length == 0)
    return;

  // This is an attempt to make the copy_array fast.
  int l2es = log2_element_size();
  size_t src_offset = arrayOopDesc::base_offset_in_bytes(element_type()) + ((size_t)src_pos << l2es);
  size_t dst_offset = arrayOopDesc::base_offset_in_bytes(element_type()) + ((size_t)dst_pos << l2es);
  ArrayAccess<ARRAYCOPY_ATOMIC>::arraycopy<void>(s, src_offset, d, dst_offset, (size_t)length << l2es);
}
```

- `src/hotspot/share/oops/objArrayKlass.cpp` 

```java
// Either oop or narrowOop depending on UseCompressedOops.
void ObjArrayKlass::do_copy(arrayOop s, size_t src_offset,
                            arrayOop d, size_t dst_offset, int length, TRAPS) {
    printf("zhenghai ->> do copy in objArrayKlass.cpp:205\n");
  if (s == d) {
    // since source and destination are equal we do not need conversion checks.
    assert(length > 0, "sanity check");
    ArrayAccess<>::oop_arraycopy(s, src_offset, d, dst_offset, length);
  } else {
    // We have to make sure all elements conform to the destination array
    Klass* bound = ObjArrayKlass::cast(d->klass())->element_klass();
    Klass* stype = ObjArrayKlass::cast(s->klass())->element_klass();
    if (stype == bound || stype->is_subtype_of(bound)) {
      // elements are guaranteed to be subtypes, so no check necessary
      ArrayAccess<ARRAYCOPY_DISJOINT>::oop_arraycopy(s, src_offset, d, dst_offset, length);
    } else {
      // slow case: need individual subtype checks
      // note: don't use obj_at_put below because it includes a redundant store check
      if (!ArrayAccess<ARRAYCOPY_DISJOINT | ARRAYCOPY_CHECKCAST>::oop_arraycopy(s, src_offset, d, dst_offset, length)) {
        ResourceMark rm(THREAD);
        stringStream ss;
        if (!bound->is_subtype_of(stype)) {
          ss.print("arraycopy: type mismatch: can not copy %s[] into %s[]",
                   stype->external_name(), bound->external_name());
        } else {
          // oop_arraycopy should return the index in the source array that
          // contains the problematic oop.
          ss.print("arraycopy: element type mismatch: can not cast one of the elements"
                   " of %s[] to the type of the destination array, %s",
                   stype->external_name(), bound->external_name());
        }
        THROW_MSG(vmSymbols::java_lang_ArrayStoreException(), ss.as_string());
      }
    }
  }
}

void ObjArrayKlass::copy_array(arrayOop s, int src_pos, arrayOop d,
                               int dst_pos, int length, TRAPS) {
  assert(s->is_objArray(), "must be obj array");

  printf("zhenghai ->> src/hotspot/share/oops/objArrayKlass.cpp:243\n");
  if (!d->is_objArray()) {
    ResourceMark rm(THREAD);
    stringStream ss;
    if (d->is_typeArray()) {
      ss.print("arraycopy: type mismatch: can not copy object array[] into %s[]",
               type2name_tab[ArrayKlass::cast(d->klass())->element_type()]);
    } else {
      ss.print("arraycopy: destination type %s is not an array", d->klass()->external_name());
    }
    THROW_MSG(vmSymbols::java_lang_ArrayStoreException(), ss.as_string());
  }

  // Check is all offsets and lengths are non negative
  if (src_pos < 0 || dst_pos < 0 || length < 0) {
    // Pass specific exception reason.
    ResourceMark rm(THREAD);
    stringStream ss;
    if (src_pos < 0) {
      ss.print("arraycopy: source index %d out of bounds for object array[%d]",
               src_pos, s->length());
    } else if (dst_pos < 0) {
      ss.print("arraycopy: destination index %d out of bounds for object array[%d]",
               dst_pos, d->length());
    } else {
      ss.print("arraycopy: length %d is negative", length);
    }
    THROW_MSG(vmSymbols::java_lang_ArrayIndexOutOfBoundsException(), ss.as_string());
  }
  // Check if the ranges are valid
  if ((((unsigned int) length + (unsigned int) src_pos) > (unsigned int) s->length()) ||
      (((unsigned int) length + (unsigned int) dst_pos) > (unsigned int) d->length())) {
    // Pass specific exception reason.
    ResourceMark rm(THREAD);
    stringStream ss;
    if (((unsigned int) length + (unsigned int) src_pos) > (unsigned int) s->length()) {
      ss.print("arraycopy: last source index %u out of bounds for object array[%d]",
               (unsigned int) length + (unsigned int) src_pos, s->length());
    } else {
      ss.print("arraycopy: last destination index %u out of bounds for object array[%d]",
               (unsigned int) length + (unsigned int) dst_pos, d->length());
    }
    THROW_MSG(vmSymbols::java_lang_ArrayIndexOutOfBoundsException(), ss.as_string());
  }

  // Special case. Boundary cases must be checked first
  // This allows the following call: copy_array(s, s.length(), d.length(), 0).
  // This is correct, since the position is supposed to be an 'in between point', i.e., s.length(),
  // points to the right of the last element.
  if (length==0) {
    return;
  }
  printf("zhenghai ->> %d  objArrayKlass.cpp:295\n",UseCompressedOops);
  if (UseCompressedOops) {
    size_t src_offset = (size_t) objArrayOopDesc::obj_at_offset<narrowOop>(src_pos);
    size_t dst_offset = (size_t) objArrayOopDesc::obj_at_offset<narrowOop>(dst_pos);
    assert(arrayOopDesc::obj_offset_to_raw<narrowOop>(s, src_offset, NULL) ==
           objArrayOop(s)->obj_at_addr<narrowOop>(src_pos), "sanity");
    assert(arrayOopDesc::obj_offset_to_raw<narrowOop>(d, dst_offset, NULL) ==
           objArrayOop(d)->obj_at_addr<narrowOop>(dst_pos), "sanity");
    do_copy(s, src_offset, d, dst_offset, length, CHECK);
  } else {
    size_t src_offset = (size_t) objArrayOopDesc::obj_at_offset<oop>(src_pos);
    size_t dst_offset = (size_t) objArrayOopDesc::obj_at_offset<oop>(dst_pos);
    assert(arrayOopDesc::obj_offset_to_raw<oop>(s, src_offset, NULL) ==
           objArrayOop(s)->obj_at_addr<oop>(src_pos), "sanity");
    assert(arrayOopDesc::obj_offset_to_raw<oop>(d, dst_offset, NULL) ==
           objArrayOop(d)->obj_at_addr<oop>(dst_pos), "sanity");
    do_copy(s, src_offset, d, dst_offset, length, CHECK);
  }
}
```

通过两源文件的名字我们大致可以猜到一个针对类型数组，一个针对对象数组。简单的阅读源码我们可以找到

typeArrayKlass调用ArrayAccess类中的静态函数arraycopy，而objArrayKlass调用的则是oop_arraycopy。

