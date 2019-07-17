java_reflection_.md

包结构

###  接口
- AnnotatedElement
- GenericDeclaration
- Member
- AnnotatedType
- AnnotatedWildcardType
- AnnotatedTypeVariable
- AnnotatedParameterizedType
- AnnotatedArrayType
- Type
- TyepVariable
- GenericArrayType
- WildcardType
- ParameterizedType

### 类
- AccessibleObject
- Field
- Executable
- Method
- Constructor

### 代理相关
- Proxy
- InvocationHandler

### 其它
- ReflectPermission
- ReflectAccess
- WeakCache
- Modifier
- Array



## Executable
### getParameterTypes 与 getGenericParameterTypes
```
    public abstract Class<?>[] ();
    public Type[] getGenericParameterTypes()
```
两个方法基本上是一样的， 不同在于getParameterTypes返回是的Class, getGenericParameterTypes 返回的是Type, 相对来说 getGenericParameterTypes 返回的内容更多（有泛型的情况下）

