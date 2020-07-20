---
layout:     post   				    # 使用的布局（不需要改）
title:      class_copyIvarList & class_copyPropertyList 				# 标题 
subtitle:   积少成多 #副标题
date:       2020-07-20				# 时间
author:     bkun 						# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - iOS
---
###class_copyIvarList
其源码，

```
Ivar *
class_copyIvarList(Class cls, unsigned int *outCount)
{
    const ivar_list_t *ivars;
    Ivar *result = nil;
    unsigned int count = 0;

    if (!cls) {
        if (outCount) *outCount = 0;
        return nil;
    }

    mutex_locker_t lock(runtimeLock);

    assert(cls->isRealized());
    // 获取类中的 data 的 class_ro_t 中的 ivars
    if ((ivars = cls->data()->ro->ivars)  &&  ivars->count) {
        result = (Ivar *)malloc((ivars->count+1) * sizeof(Ivar));
        
        for (auto& ivar : *ivars) {
            if (!ivar.offset) continue;  // anonymous bitfield
            result[count++] = &ivar;
        }
        result[count] = nil;
    }
    
    if (outCount) *outCount = count;
    return result;
}

```

1. 根据源码可知是通过 ``` data()``` 方法获取了 ``` class_rw_t```

```
//类对象
class_rw_t *data() { 
        return bits.data();
}

// class_data_bits_t
 class_rw_t* data() {
        return (class_rw_t *)(bits & FAST_DATA_MASK);
    }
```

2. ->ro->ivars 获取了 ``` class_ro_t``` 中的变量 ```ivars```

###  class_copyPropertyList
其源码，

```
objc_property_t *
class_copyPropertyList(Class cls, unsigned int *outCount)
{
    if (!cls) {
        if (outCount) *outCount = 0;
        return nil;
    }

    mutex_locker_t lock(runtimeLock);

    checkIsKnownClass(cls);
    assert(cls->isRealized());
    
    auto rw = cls->data();

    property_t **result = nil;
    unsigned int count = rw->properties.count();
    if (count > 0) {
        result = (property_t **)malloc((count + 1) * sizeof(property_t *));

        count = 0;
        for (auto& prop : rw->properties) {
            result[count++] = &prop;
        }
        result[count] = nil;
    }

    if (outCount) *outCount = count;
    return (objc_property_t *)result;
}

```

1. 和 ``` class_copyIvarList ``` 一致通过 ``` data()``` 方法获取了 ``` class_rw_t```
2. ```  rw->properties``` 获取的是 ``` class_rw_t ``` 中的 ``` properties``` 

属性其实是包含了实例变量和 setter 和 getter 方法。``` class_copyIvarList ``` 可以获取属性声明的生成的实例变量。
