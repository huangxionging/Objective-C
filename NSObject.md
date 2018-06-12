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
	struct _class_ro_t *ro; // 只读常量链表
};

// category结构体
struct _category_t {
	const char *name;
	struct _class_t *cls;
	const struct _method_list_t *instance_methods;
	const struct _method_list_t *class_methods;
	const struct _protocol_list_t *protocols;
	const struct _prop_list_t *properties;
};

