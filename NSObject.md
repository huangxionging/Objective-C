# NSObject 

NSObject 是 Foundation 的基类, 最后被编译成 C 代码
我们来看
```Objective-C
typedef struct objc_object NSObject;
typedef struct {} _objc_exc_NSObject;
#endif

struct NSObject_IMPL {
	Class isa;
};
```
Class 是 objc_class, 而 objc_class 继承自 objc_object
```C++
struct objc_object {
private:
// isa 
    isa_t isa;

public:

    // ISA() assumes this is NOT a tagged pointer object
    Class ISA();

    // getIsa() allows this to be a tagged pointer object
    Class getIsa();

    // initIsa() should be used to init the isa of new objects only.
    // If this object already has an isa, use changeIsa() for correctness.
    // initInstanceIsa(): objects with no custom RR/AWZ
    // initClassIsa(): class objects
    // initProtocolIsa(): protocol objects
    // initIsa(): other objects
    void initIsa(Class cls /*nonpointer=false*/);
    void initClassIsa(Class cls /*nonpointer=maybe*/);
    void initProtocolIsa(Class cls /*nonpointer=maybe*/);
    void initInstanceIsa(Class cls, bool hasCxxDtor);

    // changeIsa() should be used to change the isa of existing objects.
    // If this is a new object, use initIsa() for performance.
    Class changeIsa(Class newCls);

    bool hasNonpointerIsa();
    bool isTaggedPointer();
    bool isBasicTaggedPointer();
    bool isExtTaggedPointer();
    bool isClass();

    // object may have associated objects?
    bool hasAssociatedObjects();
    void setHasAssociatedObjects();

    // object may be weakly referenced?
    bool isWeaklyReferenced();
    void setWeaklyReferenced_nolock();

    // object may have -.cxx_destruct implementation?
    bool hasCxxDtor();

    // Optimized calls to retain/release methods
    id retain();
    void release();
    id autorelease();

    // Implementations of retain/release methods
    id rootRetain();
    bool rootRelease();
    id rootAutorelease();
    bool rootTryRetain();
    bool rootReleaseShouldDealloc();
    uintptr_t rootRetainCount();

    // Implementation of dealloc methods
    bool rootIsDeallocating();
    void clearDeallocating();
    void rootDealloc();

private:
    void initIsa(Class newCls, bool nonpointer, bool hasCxxDtor);

    // Slow paths for inline control
    id rootAutorelease2();
    bool overrelease_error();

#if SUPPORT_NONPOINTER_ISA
    // Unified retain count manipulation for nonpointer isa
    id rootRetain(bool tryRetain, bool handleOverflow);
    bool rootRelease(bool performDealloc, bool handleUnderflow);
    id rootRetain_overflow(bool tryRetain);
    bool rootRelease_underflow(bool performDealloc);

    void clearDeallocating_slow();

    // Side table retain count overflow for nonpointer isa
    void sidetable_lock();
    void sidetable_unlock();

    void sidetable_moveExtraRC_nolock(size_t extra_rc, bool isDeallocating, bool weaklyReferenced);
    bool sidetable_addExtraRC_nolock(size_t delta_rc);
    size_t sidetable_subExtraRC_nolock(size_t delta_rc);
    size_t sidetable_getExtraRC_nolock();
#endif

    // Side-table-only retain count
    bool sidetable_isDeallocating();
    void sidetable_clearDeallocating();

    bool sidetable_isWeaklyReferenced();
    void sidetable_setWeaklyReferenced_nolock();

    id sidetable_retain();
    id sidetable_retain_slow(SideTable& table);

    uintptr_t sidetable_release(bool performDealloc = true);
    uintptr_t sidetable_release_slow(SideTable& table, bool performDealloc = true);

    bool sidetable_tryRetain();

    uintptr_t sidetable_retainCount();
#if DEBUG
    bool sidetable_present();
#endif
};
```
```C++
struct objc_class : objc_object {
    // Class ISA;
    // 指向父类
    Class superclass;
    // 缓存 变量表
    cache_t cache;             // formerly cache pointer and vtable
    // 数据位
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

    // 可读写结构体数据链表
    class_rw_t *data() { 
        return bits.data();
    }
    void setData(class_rw_t *newData) {
        bits.setData(newData);
    }

    void setInfo(uint32_t set) {
        assert(isFuture()  ||  isRealized());
        data()->setFlags(set);
    }

    void clearInfo(uint32_t clear) {
        assert(isFuture()  ||  isRealized());
        data()->clearFlags(clear);
    }

    // set and clear must not overlap
    void changeInfo(uint32_t set, uint32_t clear) {
        assert(isFuture()  ||  isRealized());
        assert((set & clear) == 0);
        data()->changeFlags(set, clear);
    }

    bool hasCustomRR() {
        return ! bits.hasDefaultRR();
    }
    void setHasDefaultRR() {
        assert(isInitializing());
        bits.setHasDefaultRR();
    }
    void setHasCustomRR(bool inherited = false);
    void printCustomRR(bool inherited);

    bool hasCustomAWZ() {
        return ! bits.hasDefaultAWZ();
    }
    void setHasDefaultAWZ() {
        assert(isInitializing());
        bits.setHasDefaultAWZ();
    }
    void setHasCustomAWZ(bool inherited = false);
    void printCustomAWZ(bool inherited);

    bool instancesRequireRawIsa() {
        return bits.instancesRequireRawIsa();
    }
    void setInstancesRequireRawIsa(bool inherited = false);
    void printInstancesRequireRawIsa(bool inherited);

    bool canAllocNonpointer() {
        assert(!isFuture());
        return !instancesRequireRawIsa();
    }
    bool canAllocFast() {
        assert(!isFuture());
        return bits.canAllocFast();
    }


    bool hasCxxCtor() {
        // addSubclass() propagates this flag from the superclass.
        assert(isRealized());
        return bits.hasCxxCtor();
    }
    void setHasCxxCtor() { 
        bits.setHasCxxCtor();
    }

    bool hasCxxDtor() {
        // addSubclass() propagates this flag from the superclass.
        assert(isRealized());
        return bits.hasCxxDtor();
    }
    void setHasCxxDtor() { 
        bits.setHasCxxDtor();
    }


    bool isSwift() {
        return bits.isSwift();
    }


    // Return YES if the class's ivars are managed by ARC, 
    // or the class is MRC but has ARC-style weak ivars.
    bool hasAutomaticIvars() {
        return data()->ro->flags & (RO_IS_ARC | RO_HAS_WEAK_WITHOUT_ARC);
    }

    // Return YES if the class's ivars are managed by ARC.
    bool isARC() {
        return data()->ro->flags & RO_IS_ARC;
    }


#if SUPPORT_NONPOINTER_ISA
    // Tracked in non-pointer isas; not tracked otherwise
#else
    bool instancesHaveAssociatedObjects() {
        // this may be an unrealized future class in the CF-bridged case
        assert(isFuture()  ||  isRealized());
        return data()->flags & RW_INSTANCES_HAVE_ASSOCIATED_OBJECTS;
    }

    void setInstancesHaveAssociatedObjects() {
        // this may be an unrealized future class in the CF-bridged case
        assert(isFuture()  ||  isRealized());
        setInfo(RW_INSTANCES_HAVE_ASSOCIATED_OBJECTS);
    }
#endif

    bool shouldGrowCache() {
        return true;
    }

    void setShouldGrowCache(bool) {
        // fixme good or bad for memory use?
    }

    bool isInitializing() {
        return getMeta()->data()->flags & RW_INITIALIZING;
    }

    void setInitializing() {
        assert(!isMetaClass());
        ISA()->setInfo(RW_INITIALIZING);
    }

    bool isInitialized() {
        return getMeta()->data()->flags & RW_INITIALIZED;
    }

    void setInitialized();

    bool isLoadable() {
        assert(isRealized());
        return true;  // any class registered for +load is definitely loadable
    }

    IMP getLoadMethod();

    // Locking: To prevent concurrent realization, hold runtimeLock.
    bool isRealized() {
        return data()->flags & RW_REALIZED;
    }

    // Returns true if this is an unrealized future class.
    // Locking: To prevent concurrent realization, hold runtimeLock.
    bool isFuture() { 
        return data()->flags & RW_FUTURE;
    }

    bool isMetaClass() {
        assert(this);
        assert(isRealized());
        return data()->ro->flags & RO_META;
    }

    // NOT identical to this->ISA when this is a metaclass
    Class getMeta() {
        if (isMetaClass()) return (Class)this;
        else return this->ISA();
    }

    bool isRootClass() {
        return superclass == nil;
    }
    bool isRootMetaclass() {
        return ISA() == (Class)this;
    }

    const char *mangledName() { 
        // fixme can't assert locks here
        assert(this);

        if (isRealized()  ||  isFuture()) {
            return data()->ro->name;
        } else {
            return ((const class_ro_t *)data())->name;
        }
    }
    
    const char *demangledName(bool realize = false);
    const char *nameForLogging();

    // May be unaligned depending on class's ivars.
    uint32_t unalignedInstanceStart() {
        assert(isRealized());
        return data()->ro->instanceStart;
    }

    // Class's instance start rounded up to a pointer-size boundary.
    // This is used for ARC layout bitmaps.
    uint32_t alignedInstanceStart() {
        return word_align(unalignedInstanceStart());
    }

    // May be unaligned depending on class's ivars.
    uint32_t unalignedInstanceSize() {
        assert(isRealized());
        return data()->ro->instanceSize;
    }

    // Class's ivar size rounded up to a pointer-size boundary.
    uint32_t alignedInstanceSize() {
        return word_align(unalignedInstanceSize());
    }

    size_t instanceSize(size_t extraBytes) {
        size_t size = alignedInstanceSize() + extraBytes;
        // CF requires all objects be at least 16 bytes.
        if (size < 16) size = 16;
        return size;
    }

    void setInstanceSize(uint32_t newSize) {
        assert(isRealized());
        if (newSize != data()->ro->instanceSize) {
            assert(data()->flags & RW_COPIED_RO);
            *const_cast<uint32_t *>(&data()->ro->instanceSize) = newSize;
        }
        bits.setFastInstanceSize(newSize);
    }

    void chooseClassArrayIndex();

    void setClassArrayIndex(unsigned Idx) {
        bits.setClassArrayIndex(Idx);
    }

    unsigned classArrayIndex() {
        return bits.classArrayIndex();
    }

};
```
然后我们以 Class Office 为例, 简述 Class 被编译后的样子
Office 定义如下
```Objective-C
//
//  Office.h
//  PersonAndLighting
//
//  Created by 黄雄 on 2018/5/28.
//  Copyright © 2018年 黄雄. All rights reserved.
//

#import <Foundation/Foundation.h>
#import "Person.h"
#import "Light.h"

@interface Office : NSObject

@property(nonatomic, strong) NSMutableArray *persons;
@property(nonatomic, retain) Light *light;


/**
 人进入办公室

 @param person 人类对象
 */
- (void)personGoOffice:(Person *)person;

/**
 人离开办公室

 @param person 人类对象
 */
- (void)personLeaveOffice: (Person *)person;

/**
 获得灯的引用计算器数值

 @return 灯的引用计算器数值
 */
- (NSInteger) retainCountOfLight;

@end
```
```Objective-C
//
//  Office.m
//  PersonAndLighting
//
//  Created by 黄雄 on 2018/5/28.
//  Copyright © 2018年 黄雄. All rights reserved.
//

#import "Office.h"


@implementation Office

@synthesize persons = _persons, light = _light;

+ (NSString *) descriptionClass {
    return @"Office";
}

- (NSMutableArray *)persons {
    if (_persons == nil) {
        _persons = [[NSMutableArray alloc] init];
    }
    return _persons;
}

- (instancetype)init {
    if (self = [super init]) {
//        Light *light = [[Light alloc] init];
        self.light = [Light new];
//        [light release];
        NSLog(@"light: %ld", self.retainCountOfLight);
    }
    return self;
}

- (void)setLight:(Light *)light {
    [_light release];
    _light = [light retain];
}

//- (Light *)light {
//    if (_light == nil) {
//        _light = [[Light alloc] init];
//    }
//    return _light;
//}

- (void)personGoOffice:(Person *)person {
    [self.persons addObject: person];
    [self.light retain];
}

- (void)personLeaveOffice:(Person *)person {
    [self.persons removeObject: person];
    [self.light release];
}

- (NSInteger)retainCountOfLight {
    return  self.light.retainCount;
}

@end
```
然后我们使用 **clang -rewrite-objc Office.m** 命令生成 Office.cpp 然后分析源码如下:
```C++
#ifndef _REWRITER_typedef_Office
#define _REWRITER_typedef_Office
typedef struct objc_object Office; // 生成的 objc 对象
typedef struct {} _objc_exc_Office; // 执行的 Office
#endif

extern "C" unsigned long OBJC_IVAR_$_Office$_persons; // 个人变量
extern "C" unsigned long OBJC_IVAR_$_Office$_light; // 灯变量
// 实现
struct Office_IMPL {
	struct NSObject_IMPL NSObject_IVARS; // NSObject 的实现
	NSMutableArray *_persons; // 数组
	Light *_light; // 等
};

// 属性被转换成变量
// @property(nonatomic, strong) NSMutableArray *persons;
// @property(nonatomic, retain) Light *light;





// 方法被转换成了 C 语言代码
// - (void)personGoOffice:(Person *)person;
// - (void)personLeaveOffice: (Person *)person;
// - (NSInteger) retainCountOfLight;

/* @end */

// @implementation Office

// // @synthesize persons = _persons, light = _light;
// 静态方法
static void _I_Office_setPersons_(Office * self, SEL _cmd, NSMutableArray *persons) { 
    (*(NSMutableArray **)((char *)self + OBJC_IVAR_$_Office$_persons)) = persons;
}

static Light * _I_Office_light(Office * self, SEL _cmd) { 
    // 获得 light 变量
    return (*(Light **)((char *)self + OBJC_IVAR_$_Office$_light));
}
// 设置属性
extern "C" __declspec(dllimport) void objc_setProperty (id, SEL, long, id, bool, bool);

static void _I_Office_setLight_(Office * self, SEL _cmd, Light *light) {
    // 设置属性
    objc_setProperty (self, _cmd, __OFFSETOFIVAR__(struct Office, _light), (id)light, 0, 0);
}

// 获得 person 的代码
static NSMutableArray * _I_Office_persons(Office * self, SEL _cmd) {
    if ((*(NSMutableArray **)((char *)self + OBJC_IVAR_$_Office$_persons)) == __null) {
        (*(NSMutableArray **)((char *)self + OBJC_IVAR_$_Office$_persons)) = ((NSMutableArray *(*)(id, SEL))(void *)objc_msgSend)((id)((NSMutableArray *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSMutableArray"), sel_registerName("alloc")), sel_registerName("init"));
    }
    return (*(NSMutableArray **)((char *)self + OBJC_IVAR_$_Office$_persons));
}
// 初始化代码
static instancetype _I_Office_init(Office * self, SEL _cmd) {
    if (self = ((Office *(*)(__rw_objc_super *, SEL))(void *)objc_msgSendSuper)((__rw_objc_super){(id)self, (id)class_getSuperclass(objc_getClass("Office"))}, sel_registerName("init"))) {
        ((void (*)(id, SEL, Light *))(void *)objc_msgSend)((id)self, sel_registerName("setLight:"), ((Light *(*)(id, SEL))(void *)objc_msgSend)((id)((Light *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("Light"), sel_registerName("alloc")), sel_registerName("init")));
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_tj_swwc5gts7b5b30q17c4ldzg80000gn_T_Office_704682_mi_0, ((NSInteger (*)(id, SEL))(void *)objc_msgSend)((id)self, sel_registerName("retainCountOfLight")));
    }
    return self;
}

static void _I_Office_personGoOffice_(Office * self, SEL _cmd, Person *person) {
    ((void (*)(id, SEL, ObjectType _Nonnull))(void *)objc_msgSend)((id)((NSMutableArray *(*)(id, SEL))(void *)objc_msgSend)((id)self, sel_registerName("persons")), sel_registerName("addObject:"), (id _Nonnull)person);
    ((Light *(*)(id, SEL))(void *)objc_msgSend)((id)((Light *(*)(id, SEL))(void *)objc_msgSend)((id)self, sel_registerName("light")), sel_registerName("retain"));
}


static void _I_Office_personLeaveOffice_(Office * self, SEL _cmd, Person *person) {
    ((void (*)(id, SEL, ObjectType _Nonnull))(void *)objc_msgSend)((id)((NSMutableArray *(*)(id, SEL))(void *)objc_msgSend)((id)self, sel_registerName("persons")), sel_registerName("removeObject:"), (id _Nonnull)person);
    ((void (*)(id, SEL))(void *)objc_msgSend)((id)((Light *(*)(id, SEL))(void *)objc_msgSend)((id)self, sel_registerName("light")), sel_registerName("release"));
}


static NSInteger _I_Office_retainCountOfLight(Office * self, SEL _cmd) {
    return ((NSUInteger (*)(id, SEL))(void *)objc_msgSend)((id)((Light *(*)(id, SEL))(void *)objc_msgSend)((id)self, sel_registerName("light")), sel_registerName("retainCount"));
}

// @end


// 属性表
struct _prop_t {
	const char *name; // 名称
	const char *attributes; // 特征
};

struct _protocol_t; // 协议表

// 方法表
struct _objc_method {
	struct objc_selector * _cmd; // 选择器
	const char *method_type; // 方法类型
	void  *_imp; // 实现地址
};

struct _protocol_t {
	void * isa;  // NULL
	const char *protocol_name; // 协议名称
	const struct _protocol_list_t * protocol_list; // super protocols // 父协议
	const struct method_list_t *instance_methods; // 实例方法链表
	const struct method_list_t *class_methods; // 类方法链表
	const struct method_list_t *optionalInstanceMethods; // 可选的实例方法链表
	const struct method_list_t *optionalClassMethods; // 可选的类方法链表
	const struct _prop_list_t * properties;         // 属性链表
	const unsigned int size;  // sizeof(struct _protocol_t) // 占用的大小
	const unsigned int flags;  // = 0
	const char ** extendedMethodTypes; // 扩展方法类型
};

// 变量
struct _ivar_t {
	unsigned long int *offset;  // pointer to ivar offset location // 偏移量
	const char *name; // 名称
	const char *type; // 类型
	unsigned int alignment; // 布局
	unsigned int  size; // 大小
};

// 只读结构体
struct _class_ro_t {
	unsigned int flags; // 标记
	unsigned int instanceStart; // 实例开始位置
	unsigned int instanceSize; // 实例大小
	unsigned int reserved; // 保留
	const unsigned char *ivarLayout; // 变量布局
	const char *name; // 名字
	const struct _method_list_t *baseMethods; // 基本方法链表
	const struct _objc_protocol_list *baseProtocols; // 基本协议链表
	const struct _ivar_list_t *ivars; // 变量链表
	const unsigned char *weakIvarLayout; // weak 变量
	const struct _prop_list_t *properties; // 属性链表
};

// 类结构体
struct _class_t {
	struct _class_t *isa;   // ISA链表
	struct _class_t *superclass; // 父类链表
	void *cache; // 缓存
	void *vtable; // 变量表
	struct _class_ro_t *ro; // 只读常量链表
};

// category结构体
struct _category_t {
	const char *name; // 名字
	struct _class_t *cls; // 所属类
	const struct _method_list_t *instance_methods; // 实例方法
	const struct _method_list_t *class_methods; // 类方法
	const struct _protocol_list_t *protocols; // 协议
	const struct _prop_list_t *properties; // 属性
};
extern "C" __declspec(dllimport) struct objc_cache _objc_empty_cache;
#pragma warning(disable:4273)

extern "C" unsigned long int OBJC_IVAR_$_Office$_persons __attribute__ ((used, section ("__DATA,__objc_ivar"))) = __OFFSETOFIVAR__(struct Office, _persons);
extern "C" unsigned long int OBJC_IVAR_$_Office$_light __attribute__ ((used, section ("__DATA,__objc_ivar"))) = __OFFSETOFIVAR__(struct Office, _light);

// 初始化变量表
static struct /*_ivar_list_t*/ {
	unsigned int entsize;  // sizeof(struct _prop_t)
	unsigned int count;
	struct _ivar_t ivar_list[2];
} _OBJC_$_INSTANCE_VARIABLES_Office __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_ivar_t),
	2,
	{{(unsigned long int *)&OBJC_IVAR_$_Office$_persons, "_persons", "@\"NSMutableArray\"", 3, 8},
    // 关联 _persons 和 OBJC_IVAR_$_Office$_persons
	 {(unsigned long int *)&OBJC_IVAR_$_Office$_light, "_light", "@\"Light\"", 3, 8}}
     // 关联 _light 和 OBJC_IVAR_$_Office$_light
};

// 关联方法表
static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[8];
} _OBJC_$_INSTANCE_METHODS_Office __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	8,
	{{(struct objc_selector *)"persons", "@16@0:8", (void *)_I_Office_persons}, // 关联方法
	{(struct objc_selector *)"init", "@16@0:8", (void *)_I_Office_init}, // 关联方法
	{(struct objc_selector *)"personGoOffice:", "v24@0:8@16", (void *)_I_Office_personGoOffice_}, // 
	{(struct objc_selector *)"personLeaveOffice:", "v24@0:8@16", (void *)_I_Office_personLeaveOffice_},
	{(struct objc_selector *)"retainCountOfLight", "q16@0:8", (void *)_I_Office_retainCountOfLight},
	{(struct objc_selector *)"setPersons:", "v24@0:8@16", (void *)_I_Office_setPersons_},
	{(struct objc_selector *)"light", "@16@0:8", (void *)_I_Office_light},
	{(struct objc_selector *)"setLight:", "v24@0:8@16", (void *)_I_Office_setLight_}}
};

static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[1];
} _OBJC_$_CLASS_METHODS_Office __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	1,
	{{(struct objc_selector *)"descriptionClass", "@16@0:8", (void *)_C_Office_descriptionClass}}
};

// 关联属性表
static struct /*_prop_list_t*/ {
	unsigned int entsize;  // sizeof(struct _prop_t)
	unsigned int count_of_properties;
	struct _prop_t prop_list[2];
} _OBJC_$_PROP_LIST_Office __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_prop_t),
	2,
	{{"persons","T@\"NSMutableArray\",&,N,V_persons"},
	{"light","T@\"Light\",&,N,V_light"}}
};

// 定义 _OBJC_METACLASS_RO_$_Office (_class_ro_t 类型)
static struct _class_ro_t _OBJC_METACLASS_RO_$_Office __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	1, sizeof(struct _class_t), sizeof(struct _class_t), 
	(unsigned int)0, 
	0, 
	"Office",
	(const struct _method_list_t *)&_OBJC_$_CLASS_METHODS_Office,
	0, 
	0, 
	0, 
	0, 
};

// 定义 _OBJC_CLASS_RO_$_Office 类型
static struct _class_ro_t _OBJC_CLASS_RO_$_Office __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	0, 
    __OFFSETOFIVAR__(struct Office, _persons), 
    sizeof(struct Office_IMPL), // 实现
	(unsigned int)0, 
	0, 
	"Office",
	(const struct _method_list_t *)&_OBJC_$_INSTANCE_METHODS_Office, // 前面的实例方法表
	0, 
	(const struct _ivar_list_t *)&_OBJC_$_INSTANCE_VARIABLES_Office, // 前面的实例变量表
	0, 
	(const struct _prop_list_t *)&_OBJC_$_PROP_LIST_Office, // 前面的属性表
};

extern "C" __declspec(dllimport) struct _class_t OBJC_METACLASS_$_NSObject;

// 定义 OBJC_METACLASS_$_Office 元类
extern "C" __declspec(dllexport) struct _class_t OBJC_METACLASS_$_Office __attribute__ ((used, section ("__DATA,__objc_data"))) = {
	0, // &OBJC_METACLASS_$_NSObject,
	0, // &OBJC_METACLASS_$_NSObject,
	0, // (void *)&_objc_empty_cache,
	0, // unused, was (void *)&_objc_empty_vtable,
	&_OBJC_METACLASS_RO_$_Office, // 前面的只读元类
};

extern "C" __declspec(dllimport) struct _class_t OBJC_CLASS_$_NSObject;

// 定义 Office 类
extern "C" __declspec(dllexport) struct _class_t OBJC_CLASS_$_Office __attribute__ ((used, section ("__DATA,__objc_data"))) = {
	0, // &OBJC_METACLASS_$_Office,
	0, // &OBJC_CLASS_$_NSObject,
	0, // (void *)&_objc_empty_cache,
	0, // unused, was (void *)&_objc_empty_vtable,
	&_OBJC_CLASS_RO_$_Office, // 前面的只读类
};

// 设置方法
static void OBJC_CLASS_SETUP_$_Office(void ) {
	OBJC_METACLASS_$_Office.isa = &OBJC_METACLASS_$_NSObject;
	OBJC_METACLASS_$_Office.superclass = &OBJC_METACLASS_$_NSObject;
	OBJC_METACLASS_$_Office.cache = &_objc_empty_cache;
	OBJC_CLASS_$_Office.isa = &OBJC_METACLASS_$_Office;
	OBJC_CLASS_$_Office.superclass = &OBJC_CLASS_$_NSObject;
	OBJC_CLASS_$_Office.cache = &_objc_empty_cache;
}

#pragma section(".objc_inithooks$B", long, read, write)
// 类设置
__declspec(allocate(".objc_inithooks$B")) static void *OBJC_CLASS_SETUP[] = {
	(void *)&OBJC_CLASS_SETUP_$_Office,
};
// 
static struct _class_t *L_OBJC_LABEL_CLASS_$ [1] __attribute__((used, section ("__DATA, __objc_classlist,regular,no_dead_strip")))= {
	&OBJC_CLASS_$_Office,
};
static struct IMAGE_INFO { unsigned version; unsigned flag; } _OBJC_IMAGE_INFO = { 0, 2 };
```

从以上代码可以看出
Office 最后变成了 结构体 objc_object Office 以及结构体 OBJC_METACLASS_\$_Office 和 OBJC_CLASS\_$_Office
这三个结构体构成了 Office 类

OBJC_CLASS_SETUP 是一个指针数组, 元素指向 OBJC_CLASS_SETUP_$_Office 这个函数的地址
OBJC_CLASS_SETUP\_\$_Office 这个函数会被调用
```C++
static void OBJC_CLASS_SETUP_$_Office(void) {
    // 设置元类的 isa 指针指向 NSObject 的元类
    OBJC_METACLASS_$_Office.isa = &OBJC_METACLASS_$_NSObject;
    // 设置元类的父类. 还是指向 NSObject 的元类
    OBJC_METACLASS_$_Office.superclass = &OBJC_METACLASS_$_NSObject;
    // 设置元类的方法缓存指针, 主要缓存类方法
    OBJC_METACLASS_$_Office.cache = &_objc_empty_cache;
    // 设置类的 isa 指针指向本类的元类
    OBJC_CLASS_$_Office.isa = &OBJC_METACLASS_$_Office;
    // 设置本类的父类指针指向 NSObject 的类
    OBJC_CLASS_$_Office.superclass = &OBJC_CLASS_$_NSObject;
    // 设置类的缓存, 这个主要会缓存实例方法
    OBJC_CLASS_$_Office.cache = &_objc_empty_cache;
}

// _objc_empty_cache 是 objc_cache 结构体, 定义如下
OBJC_EXPORT struct objc_cache _objc_empty_cache
struct objc_cache {
    // 
    unsigned int mask /* total = mask + 1 */                 OBJC2_UNAVAILABLE;
    unsigned int occupied                                    OBJC2_UNAVAILABLE;
    // 存储方法的结构体指针 bucket是的元素是指向
    Method _Nullable buckets[1]                              OBJC2_UNAVAILABLE;
};
// Method 是结构体指针
typedef struct method_t *Method;
struct method_t {
    SEL name; // 方法名称
    const char *types; // 类型
    IMP imp; // 实现

    // 排序
    struct SortBySELAddress: public std::binary_function<const method_t&,const method_t&, bool>
    {
        bool operator() (const method_t& lhs, const method_t& rhs)
        { 
            return lhs.name < rhs.name; 
        }
    };
};

```
OBJC_METACLASS_$_Office 的定义
```C++
struct _class_t OBJC_METACLASS_$_Office = {
    0, // &OBJC_METACLASS_$_NSObject,
    0, // &OBJC_METACLASS_$_NSObject,
    0, // (void *)&_objc_empty_cache,
    0, // unused, was (void *)&_objc_empty_vtable,
    &_OBJC_METACLASS_RO_$_Office,
};
```
其第一个字段 isa 指针同类型的链表, 永远指向根类 OBJC_METACLAS\_\$_NSObject
第二个字段 superclass, 则指向其父类的 metaClass, 这里还是 OBJC_METACLAS\_\$_NSObject
第三个字段 cache 用于缓存方法的, 暂时也为 空
第四个字段 vtable 变量表为保留字段空
第五个字段 ro 指向 _OBJC_METACLASS_RO_$_Office 这个只读链表
```C++
// 类结构体
struct _class_t {
    struct _class_t *isa;   // ISA链表
    struct _class_t *superclass; // 父类链表
    void *cache; // 缓存
    void *vtable; // 变量表
    struct _class_ro_t *ro; // 只读常量链表
};
```
而_OBJC_METACLASS_RO_$_Office定义如下
```C++ 
static struct _class_ro_t _OBJC_METACLASS_RO_$_Office  = {
	1, 
    sizeof(struct _class_t),
    sizeof(struct _class_t), 
	(unsigned int)0, 
	0, 
	"Office",
	(const struct _method_list_t *)&_OBJC_$_CLASS_METHODS_Office,
	0, 
	0, 
	0, 
	0, 
};
// 只读结构体
struct _class_ro_t {
    unsigned int flags; // 标记
    unsigned int instanceStart; // 实例开始位置
    unsigned int instanceSize; // 实例大小
    unsigned int reserved; // 保留
    const unsigned char *ivarLayout; // 变量布局
    const char *name; // 名字
    const struct _method_list_t *baseMethods; // 基本方法链表
    const struct _objc_protocol_list *baseProtocols; // 基本协议链表
    const struct _ivar_list_t *ivars; // 变量链表
    const unsigned char *weakIvarLayout; // weak 变量
    const struct _prop_list_t *properties; // 属性链表
};
```
从以上代码看出 _OBJC_METACLASS_RO_$_Office
第一个字段 flags 标记为1
第二个字段 instanceStart, 开始位置为一个 _class_t 的大小
第三个字段 instanceSize, 大小占一个 _class_t 的大小
第四个字段 reserved 值为 0
第五个字段 ivarLayout 值为 0
第六个字段 name 表示类名, 这里值为 "Office"
第七个字段 baseMethods 为_OBJC\_\$_CLASS_METHODS_Office, 存的是类方法,
第八个字段 baseProtocols 协议为0, 表示 Office 元类没有遵守协议
第九个字段 ivars 变量表为空, 说明元类变量表尾为空
第十个字段 weakIvarLayout weak 变量布局为空
第十一个字段 properties 属性链表为空
以上是 Office metaClass

再来看 OBJC_CLASS_$_Office
isa 指向自己的 metaClass
supperClass 指向父类的 class
缓存和变量表为空
最后还剩 _class_ro_t *ro指向 _OBJC_CLASS_RO\_\$_Office
```C++
struct _class_t OBJC_CLASS_$_Office = {
	0, // &OBJC_METACLASS_$_Office,
	0, // &OBJC_CLASS_$_NSObject,
	0, // (void *)&_objc_empty_cache,
	0, // unused, was (void *)&_objc_empty_vtable,
	&_OBJC_CLASS_RO_$_Office,
};

static struct _class_ro_t _OBJC_CLASS_RO_$_Office = {
	0, 
    __OFFSETOFIVAR__(struct Office, _persons), 
    sizeof(struct Office_IMPL), 
	(unsigned int)0, 
	0, 
	"Office",
	(const struct _method_list_t *)&_OBJC_$_INSTANCE_METHODS_Office,
	(const struct _objc_protocol_list *)&_OBJC_CLASS_PROTOCOLS_$_Office,
	(const struct _ivar_list_t *)&_OBJC_$_INSTANCE_VARIABLES_Office,
	0, 
	(const struct _prop_list_t *)&_OBJC_$_PROP_LIST_Office,
};
```
从以上代码看出 _OBJC_CLASS_RO_$_Office
第一个字段 flags 标记为0
第二个字段 instanceStart, 起始位置为 偏移量为 objc_object Office的 _persons 变量位置, 这个根据变量定义顺序来决定的.
第三个字段 instanceSize, 大小占一个 Office_IMPL 的大小, 也就是类实现的大小
第四个字段 reserved 值为 0
第五个字段 ivarLayout 值为 0
第六个字段 name 表示类名, 这里值为 "Office"            
第七个字段 baseMethods 为_OBJC\_\$_INSTANCE_METHODS_Office, 存的是实例方法,
第八个字段 baseProtocols _OBJC_CLASS_PROTOCOLS\_\$_Office, 遵守的协议
第九个字段 ivars _OBJC\_\$_INSTANCE_VARIABLES_Office 表示变量链表
第十个字段 weakIvarLayout weak 变量布局为空
第十一个字段 properties _OBJC\_\$_PROP_LIST_Office 表示属性链表
以上是 Office metaClass